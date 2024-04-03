# FastJson代码审计

## *DefaultJSONParser#parseObject()*

```java
/**
 * 解析 JSON 对象
 * @param object 待解析的 JSON 对象
 * @param fieldName 字段名称
 * @return 解析后的 JSON 对象
 */
public final Object parseObject(final Map object, Object fieldName) {
    final JSONLexer lexer = this.lexer;
    
    // 如果当前 token 是 NULL，则返回 null
    if (lexer.token() == JSONToken.NULL) {
        lexer.nextToken();
        return null;
    }
    
    // 如果当前 token 是右大括号，则返回对象
    if (lexer.token() == JSONToken.RBRACE) {
        lexer.nextToken();
        return object;
    }

    // 如果当前 token 是空字符串，则返回对象
    if (lexer.token() == JSONToken.LITERAL_STRING && lexer.stringVal().length() == 0) {
        lexer.nextToken();
        return object;
    }

    // 如果当前 token 不是左大括号也不是逗号，则抛出语法错误异常
    if (lexer.token() != JSONToken.LBRACE && lexer.token() != JSONToken.COMMA) {
        throw new JSONException("syntax error, expect {, actual " + lexer.tokenName() + ", " + lexer.info());
    }

    ParseContext context = this.context;
    try {
        Map map = object instanceof JSONObject ? ((JSONObject) object).getInnerMap() : object;

        boolean setContextFlag = false;
        for (;;) {
            // 跳过空白字符
            lexer.skipWhitespace();
            char ch = lexer.getCurrent();
            if (lexer.isEnabled(Feature.AllowArbitraryCommas)) {
                // 如果允许任意逗号，则跳过所有逗号
                while (ch == ',') {
                    lexer.next();
                    lexer.skipWhitespace();
                    ch = lexer.getCurrent();
                }
            }

            boolean isObjectKey = false;
            Object key;
            if (ch == '"') {
                // 如果当前字符是双引号，则扫描字符串作为键
                key = lexer.scanSymbol(symbolTable, '"');
                lexer.skipWhitespace();
                ch = lexer.getCurrent();
                if (ch != ':') {
                    throw new JSONException("expect ':' at " + lexer.pos() + ", name " + key);
                }
            } else if (ch == '}') {
                // 如果当前字符是右大括号，则结束解析，返回对象
                lexer.next();
                lexer.resetStringPosition();
                lexer.nextToken();

                if (!setContextFlag) {
                    if (this.context != null && fieldName == this.context.fieldName && object == this.context.object) {
                        context = this.context;
                    } else {
                        ParseContext contextR = setContext(object, fieldName);
                        if (context == null) {
                            context = contextR;
                        }
                        setContextFlag = true;
                    }
                }

                return object;
            } else if (ch == '\'') {
                // 如果当前字符是单引号，则扫描字符串作为键
                if (!lexer.isEnabled(Feature.AllowSingleQuotes)) {
                    throw new JSONException("syntax error");
                }

                key = lexer.scanSymbol(symbolTable, '\'');
                lexer.skipWhitespace();
                ch = lexer.getCurrent();
                if (ch != ':') {
                    throw new JSONException("expect ':' at " + lexer.pos());
                }
            } else if (ch == EOI) {
                // 如果当前字符是结束符，则抛出语法错误异常
                throw new JSONException("syntax error");
            } else if (ch == ',') {
                // 如果当前字符是逗号，则抛出语法错误异常
                throw new JSONException("syntax error");
            } else if ((ch >= '0' && ch <= '9') || ch == '-') {
                // 如果当前字符是数字或负号，则扫描数字作为键
                lexer.resetStringPosition();
                lexer.scanNumber();
                try {
                    if (lexer.token() == JSONToken.LITERAL_INT) {
                        key = lexer.integerValue();
                    } else {
                        key = lexer.decimalValue(true);
                    }
                    if (lexer.isEnabled(Feature.NonStringKeyAsString)) {
                        key = key.toString();
                    }
                } catch (NumberFormatException e) {
                    throw new JSONException("parse number key error" + lexer.info());
                }
                ch = lexer.getCurrent();
                if (ch != ':') {
                    throw new JSONException("parse number key error" + lexer.info());
                }
            } else if (ch == '{' || ch == '[') {
                // 如果当前字符是左大括号或左中括号，则递归解析
                lexer.nextToken();
                key = parse();
                isObjectKey = true;
            } else {
                // 如果当前字符是其他字符，则尝试扫描非引号字段名
                if (!lexer.isEnabled(Feature.AllowUnQuotedFieldNames)) {
                    throw new JSONException("syntax error");
                }

                key = lexer.scanSymbolUnQuoted(symbolTable);
                lexer.skipWhitespace();
                ch = lexer.getCurrent();
                if (ch != ':') {
                    throw new JSONException("expect ':' at " + lexer.pos() + ", actual " + ch);
                }
            }

            // 如果不是对象键，则跳过当前字符并跳过空白字符
            if (!isObjectKey) {
                lexer.next();
                lexer.skipWhitespace();
            }

            ch = lexer.getCurrent();

            lexer.resetStringPosition();

            // 如果键是默认类型键且不禁用特殊键检测，则处理默认类型键
            if (key == JSON.DEFAULT_TYPE_KEY
                    && !lexer.isEnabled(Feature.DisableSpecialKeyDetect)) {
                // 扫描类型名
                String typeName = lexer.scanSymbol(symbolTable, '"');

                // 如果禁用自动类型忽略，则跳过当前循环
                if (lexer.isEnabled(Feature.IgnoreAutoType)) {
                    continue;
                }

                Class<?> clazz = null;
                if (object != null
                        && object.getClass().getName().equals(typeName)) {
                    clazz = object.getClass();
                } else {
                    clazz = config.checkAutoType(typeName, null, lexer.getFeatures());
                }

                // 如果未找到类，则将类型名添加到对象中并继续下一次循环
                if (clazz == null) {
                    map.put(JSON.DEFAULT_TYPE_KEY, typeName);
                    continue;
                }

                // 跳过逗号
                lexer.nextToken(JSONToken.COMMA);

                // 如果当前 token 是右大括号，则创建实例并返回
                if (lexer.token() == JSONToken.RBRACE) {
                    lexer.nextToken(JSONToken.COMMA);
                    try {
                        Object instance = null;
                        ObjectDeserializer deserializer = this.config.getDeserializer(clazz);
                        if (deserializer instanceof JavaBeanDeserializer) {
                            JavaBeanDeserializer javaBeanDeserializer = (JavaBeanDeserializer) deserializer;
                            instance = javaBeanDeserializer.createInstance(this, clazz);
                            
                            for (Object o : map.entrySet()) {
                                Map.Entry entry = (Map.Entry) o;
                                Object entryKey = entry.getKey();
                                if (entryKey instanceof String) {
                                    FieldDeserializer fieldDeserializer = javaBeanDeserializer.getFieldDeserializer((String) entryKey);
                                    if (fieldDeserializer != null) {
                                        fieldDeserializer.setValue(instance, entry.getValue());
                                    }
                                }
                            }
                        }

                        if (instance == null) {
                            if (clazz == Cloneable.class) {
                                instance = new HashMap();
                            } else if ("java.util.Collections$EmptyMap".equals(typeName)) {
                                instance = Collections.emptyMap();
                            } else {
                                instance = clazz.newInstance();
                            }
                        }

                        return instance;
                    } catch (Exception e) {
                        throw new JSONException("create instance error", e);
                    }
                }
                
                this.setResolveStatus(TypeNameRedirect);

                if (this.context != null
                        && fieldName != null
                        && !(fieldName instanceof Integer)
                        && !(this.context.fieldName instanceof Integer)) {
                    this.popContext();
                }
                
                // 如果对象不为空，则将其转换为指定类型并解析
                if (object.size() > 0) {
                    Object newObj = TypeUtils.cast(object, clazz, this.config);
                    this.parseObject(newObj);
                    return newObj;
                }

                // 否则，根据类型创建反序列化器并返回对象
                ObjectDeserializer deserializer = config.getDeserializer(clazz);
                Class deserClass = deserializer.getClass();
                if (JavaBeanDeserializer.class.isAssignableFrom(deserClass)
                        && deserClass != JavaBeanDeserializer.class
                        && deserClass != ThrowableDeserializer.class) {
                    this.setResolveStatus(NONE);
                }
                Object obj = deserializer.deserialze(this, clazz, fieldName);
                return obj;
            }

            // 如果键是引用键且上下文不为空且不禁用特殊键检测，则处理引用键
            if (key == "$ref"
                    && context != null
                    && !lexer.isEnabled(Feature.DisableSpecialKeyDetect)) {
                lexer.nextToken(JSONToken.LITERAL_STRING);
                if (lexer.token() == JSONToken.LITERAL_STRING) {
                    String ref = lexer.stringVal();
                    lexer.nextToken(JSONToken.RBRACE);

                    Object refValue = null;
                    if ("@".equals(ref)) {
                        if (this.context != null) {
                            ParseContext thisContext = this.context;
                            Object thisObj = thisContext.object;
                            if (thisObj instanceof Object[] || thisObj instanceof Collection<?>) {
                                refValue = thisObj;
                            } else if (thisContext.parent != null) {
                                refValue = thisContext.parent.object;
                            }
                        }
                    } else if ("..".equals(ref)) {
                        if (context.object != null) {
                            refValue = context.object;
                        } else {
                            addResolveTask(new ResolveTask(context, ref));
                            setResolveStatus(DefaultJSONParser.NeedToResolve);
                        }
                    } else if ("$".equals(ref)) {
                        ParseContext rootContext = context;
                        while (rootContext.parent != null) {
                            rootContext = rootContext.parent;
                        }

                        if (rootContext.object != null) {
                            refValue = rootContext.object;
                        } else {
                            addResolveTask(new ResolveTask(rootContext, ref));
                            setResolveStatus(DefaultJSONParser.NeedToResolve);
                        }
                    } else {
                        addResolveTask(new ResolveTask(context, ref));
                        setResolveStatus(DefaultJSONParser.NeedToResolve);
                    }

                    if (lexer.token() != JSONToken.RBRACE) {
                        throw new JSONException("syntax error");
                    }
                    lexer.nextToken(JSONToken.COMMA);

                    return refValue;
                } else {
                    throw new JSONException("illegal ref, " + JSONToken.name(lexer.token()));
                }
            }

            // 如果未设置上下文标志，则设置上下文
            if (!setContextFlag) {
                if (this.context != null && fieldName == this.context.fieldName && object == this.context.object) {
                    context = this.context;
                } else {
                    ParseContext contextR = setContext(object, fieldName);
                    if (context == null) {
                        context = contextR;
                    }
                    setContextFlag = true;
                }
            }

            if (object.getClass() == JSONObject.class) {
                if (key == null) {
                    key = "null";
                }
            }

            Object value;
            if (ch == '"') {
                // 如果当前字符是双引号，则扫描字符串作为值
                lexer.scanString();
                String strValue = lexer.stringVal();
                value = strValue;

                // 如果启用 ISO8601 日期格式，则尝试解析为日期
                if (lexer.isEnabled(Feature.AllowISO8601DateFormat)) {
                    JSONScanner iso8601Lexer = new JSONScanner(strValue);
                    if (iso8601Lexer.scanISO8601DateIfMatch()) {
                        value = iso8601Lexer.getCalendar().getTime();
                    }
                    iso8601Lexer.close();
                }

                map.put(key, value);
            } else if (ch >= '0' && ch <= '9' || ch == '-') {
                // 如果当前字符是数字或负号，则扫描数字作为值
                lexer.scanNumber();
                if (lexer.token() == JSONToken.LITERAL_INT) {
                    value = lexer.integerValue();
                } else {
                    value = lexer.decimalValue(lexer.isEnabled(Feature.UseBigDecimal));
                }

                map.put(key, value);
            } else if (ch == '[') {
                // 如果当前字符是左中括号，则解析数组
                lexer.nextToken();
                JSONArray list = new JSONArray();
                final boolean parentIsArray = fieldName != null && fieldName.getClass() == Integer.class;
                if (fieldName == null) {
                    this.setContext(context);
                }
                this.parseArray(list, key);
                if (lexer.isEnabled(Feature.UseObjectArray)) {
                    value = list.toArray();
                } else {
                    value = list;
                }
                map.put(key, value);
                if (lexer.token() == JSONToken.RBRACE) {
                    lexer.nextToken();
                    return object;
                } else if (lexer.token() == JSONToken.COMMA) {
                    continue;
                } else {
                    throw new JSONException("syntax error");
                }
            } else if (ch == '{') {
                // 如果当前字符是左大括号，则解析对象
                lexer.nextToken();
                final boolean parentIsArray = fieldName != null && fieldName.getClass() == Integer.class;
                Map input;
                if (lexer.isEnabled(Feature.CustomMapDeserializer)) {
                    MapDeserializer mapDeserializer = (MapDeserializer) config.getDeserializer(Map.class);
                    input = mapDeserializer.createMap(Map.class);
                } else {
                    input = new JSONObject(lexer.isEnabled(Feature.OrderedField));
                }
                ParseContext ctxLocal = null;
                if (!parentIsArray) {
                    ctxLocal = setContext(context, input, key);
                }
                Object obj = null;
                boolean objParsed = false;
                if (fieldTypeResolver != null) {
                    String resolveFieldName = key != null ? key.toString() : null;
                    Type fieldType = fieldTypeResolver.resolve(object, resolveFieldName);
                    if (fieldType != null) {
                        ObjectDeserializer fieldDeser = config.getDeserializer(fieldType);
                        obj = fieldDeser.deserialze(this, fieldType, key);
                        objParsed = true;
                    }
                }
                if (!objParsed) {
                    obj = this.parseObject(input, key);
                }
                if (ctxLocal != null && input != obj) {
                    ctxLocal.object = object;
                }
                if (key != null) {
                    checkMapResolve(object, key.toString());
                }
                map.put(key, obj);
                if (parentIsArray) {
                    setContext(obj, key);
                }
                if (lexer.token() == JSONToken.RBRACE) {
                    lexer.nextToken();
                    setContext(context);
                    return object;
                } else if (lexer.token() == JSONToken.COMMA) {
                    if (parentIsArray) {
                        this.popContext();
                    } else {
                        this.setContext(context);
                    }
                    continue;
                } else {
                    throw new JSONException("syntax error, " + lexer.tokenName());
                }
            } else {
                // 其他情况，解析值并添加到对象中
                lexer.nextToken();
                value = parse();
                map.put(key, value);
                if (lexer.token() == JSONToken.RBRACE) {
                    lexer.nextToken();
                    return object;
                } else if (lexer.token() == JSONToken.COMMA) {
                    continue;
                } else {
                    throw new JSONException("syntax error, position at " + lexer.pos() + ", name " + key);
                }
            }

            // 跳过空白字符并获取下一个字符
            lexer.skipWhitespace();
            ch = lexer.getCurrent();
            if (ch == ',') {
                // 如果当前字符是逗号，则继续下一次循环
                lexer.next();
                continue;
            } else if (ch == '}') {
                // 如果当前字符是右大括号，则设置上下文并返回对象
                lexer.next();
                lexer.resetStringPosition();
                lexer.nextToken();
                this.setContext(value, key);
                return object;
            } else {
                // 其他情况，抛出语法错误异常
                throw new JSONException("syntax error, position at " + lexer.pos() + ", name " + key);
            }
        }
    } finally {
        // 恢复上下文
        this.setContext(context);
    }
}
```