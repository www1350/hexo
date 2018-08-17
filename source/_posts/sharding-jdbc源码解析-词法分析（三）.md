---
title: sharding-jdbc源码解析-词法分析（三）
tags:
  - sharding-jdbc
categories:
  - 中间件
  - 数据库
abbrlink: 87f178fd
date: 2018-05-10 00:16:35
---

## 基本概念

先普及几个词汇

Lexer： 词法分析器。Lexical analyzer，简称Lexer

Literals ：字面量

Symbol： 词法符号

Dictionary： 字典

tokenize： 标记化

lexeme： 词素。词素是组成编程语言的最小的有意义的单元实体。生成的词素最后会组成一个token列表，每一个token都包含一个lexeme

Token: 标记。一个字符串，是构成[源代码](https://zh.wikipedia.org/wiki/%E6%BA%90%E4%BB%A3%E7%A0%81)的最小单位。从输入字符流中生成标记的过程叫作**标记化**（tokenization），在这个过程中，词法分析器还会对标记进行分类。

### LexerEngine

词法分析引擎

```java
@RequiredArgsConstructor
public final class LexerEngine {
    
    private final Lexer lexer;
    
    /**
     * Get input string.
     * 
     * @return inputted string
     */
    public String getInput() {
        return lexer.getInput();
    }
    
    /**
     * Analyse next token.
     */
    public void nextToken() {
        lexer.nextToken();
    }
    
    /**
     * Get current token.
     * 
     * @return current token
     */
    public Token getCurrentToken() {
        return lexer.getCurrentToken();
    }
    
    /**
     * 跳过括号内所有令牌,并返回括号内内容
     *
     * @param sqlStatement SQL statement
     * @return skipped string
     */
    public String skipParentheses(final SQLStatement sqlStatement) {
        StringBuilder result = new StringBuilder("");
        int count = 0;
        //'('开头
        if (Symbol.LEFT_PAREN == lexer.getCurrentToken().getType()) {
            //括号后的下标
            final int beginPosition = lexer.getCurrentToken().getEndPosition();
            //'('
            result.append(Symbol.LEFT_PAREN.getLiterals());
            //接着读取
            lexer.nextToken();
            while (true) {
                //如果是'?'
                if (equalAny(Symbol.QUESTION)) {
                    //参数下标加1
                    sqlStatement.increaseParametersIndex();
                }
                //结束或者')'跳出
                if (Assist.END == lexer.getCurrentToken().getType() || (Symbol.RIGHT_PAREN == lexer.getCurrentToken().getType() && 0 == count)) {
                    break;
                }
                //还是'('记下层数
                if (Symbol.LEFT_PAREN == lexer.getCurrentToken().getType()) {
                    count++;
                //')'
                } else if (Symbol.RIGHT_PAREN == lexer.getCurrentToken().getType()) {
                    count--;
                }
                //接着读取
                lexer.nextToken();
            }
            //括号里的都加进去
            result.append(lexer.getInput().substring(beginPosition, lexer.getCurrentToken().getEndPosition()));
            //跳过')'
            lexer.nextToken();
        }
        return result.toString();
    }
    
    /**
     * Assert current token type should equals input token and go to next token type.
     *
     * @param tokenType token type
     */
    public void accept(final TokenType tokenType) {
        if (lexer.getCurrentToken().getType() != tokenType) {
            throw new SQLParsingException(lexer, tokenType);
        }
        lexer.nextToken();
    }
    
    /**
     * Adjust current token equals one of input tokens or not.
     *
     * @param tokenTypes to be adjusted token types
     * @return current token equals one of input tokens or not
     */
    public boolean equalAny(final TokenType... tokenTypes) {
        for (TokenType each : tokenTypes) {
            if (each == lexer.getCurrentToken().getType()) {
                return true;
            }
        }
        return false;
    }
    
    /**
     * Skip current token if equals one of input tokens.
     *
     * @param tokenTypes to be adjusted token types
     * @return skipped current token or not
     */
    public boolean skipIfEqual(final TokenType... tokenTypes) {
        if (equalAny(tokenTypes)) {
            lexer.nextToken();
            return true;
        }
        return false;
    }
    
    /**
     * Skip all input tokens.
     *
     * @param tokenTypes to be skipped token types
     */
    public void skipAll(final TokenType... tokenTypes) {
        Set<TokenType> tokenTypeSet = Sets.newHashSet(tokenTypes);
        while (tokenTypeSet.contains(lexer.getCurrentToken().getType())) {
            lexer.nextToken();
        }
    }
    
    /**
     * Skip until one of input tokens.
     *
     * @param tokenTypes to be skipped untiled token types
     */
    public void skipUntil(final TokenType... tokenTypes) {
        Set<TokenType> tokenTypeSet = Sets.newHashSet(tokenTypes);
        tokenTypeSet.add(Assist.END);
        while (!tokenTypeSet.contains(lexer.getCurrentToken().getType())) {
            lexer.nextToken();
        }
    }
    
    /**
     * Throw unsupported exception if current token equals one of input tokens.
     * 
     * @param tokenTypes to be adjusted token types
     */
    public void unsupportedIfEqual(final TokenType... tokenTypes) {
        if (equalAny(tokenTypes)) {
            throw new SQLParsingUnsupportedException(lexer.getCurrentToken().getType());
        }
    }
    
    /**
     * Throw unsupported exception if current token not equals one of input tokens.
     *
     * @param tokenTypes to be adjusted token types
     */
    public void unsupportedIfNotSkip(final TokenType... tokenTypes) {
        if (!skipIfEqual(tokenTypes)) {
            throw new SQLParsingUnsupportedException(lexer.getCurrentToken().getType());
        }
    }
    
    /**
     * Get database type.
     * 
     * @return database type
     */
    public DatabaseType getDatabaseType() {
        if (lexer instanceof MySQLLexer) {
            return DatabaseType.MySQL;
        }
        if (lexer instanceof OracleLexer) {
            return DatabaseType.Oracle;
        }
        if (lexer instanceof SQLServerLexer) {
            return DatabaseType.SQLServer;
        }
        if (lexer instanceof PostgreSQLLexer) {
            return DatabaseType.PostgreSQL;
        }
        throw new UnsupportedOperationException(String.format("Cannot support lexer class: %s", lexer.getClass().getCanonicalName()));
    }
}
```

### Lexer

![image](https://user-images.githubusercontent.com/7789698/39862320-081c0a18-5476-11e8-8d52-2477519c9dd7.png)

```java
@RequiredArgsConstructor
public class Lexer {
    //sql
    @Getter
    private final String input;
    //
    private final Dictionary dictionary;
    //偏移量，从0开始
    private int offset;
    
    @Getter
    private Token currentToken;
}
```

```java
  /**
     * Analyse next token.
     */
    public final void nextToken() {
        //跳过空白字符、注释
        skipIgnoredToken();
        // 
        //判断变量，MySQLLexer是'@'开头
        if (isVariableBegin()) {
            //扫描变量
            currentToken = new Tokenizer(input, dictionary, offset).scanVariable();
            //mysql没有Nchar。sqlserver有。不详细展开，就是扫描nchar的
        } else if (isNCharBegin()) {
            currentToken = new Tokenizer(input, dictionary, ++offset).scanChars();
            //字母、'`'、'_'、'$' 开头的
        } else if (isIdentifierBegin()) {
            currentToken = new Tokenizer(input, dictionary, offset).scanIdentifier();
            //'0x'开头，十六进制
        } else if (isHexDecimalBegin()) {
            //扫描十六进制
            currentToken = new Tokenizer(input, dictionary, offset).scanHexDecimal();
           //数字开头、非标识符+'.'+数字、"-."、'-'+数字
        } else if (isNumberBegin()) {
            //扫描数字
            currentToken = new Tokenizer(input, dictionary, offset).scanNumber();
            //符号'(' 、')' 、'[' 、 ']'、'{'、'}'、 '+' 、 '-' 、 '*' 、 '/' 、'%' 、 '^' 、'='、 '>' 、'<' 、 '~' 、'!'、 '?'、 '&' 、'|' 、'.'、':' 、 '#'、 ',' 、 ';' 
        } else if (isSymbolBegin()) {
            //扫描符号
            currentToken = new Tokenizer(input, dictionary, offset).scanSymbol();
           //  '\'' 或'\"'开头的字符
        } else if (isCharsBegin()) {
            //扫描单引号'或双引号"括起来的东西
            currentToken = new Tokenizer(input, dictionary, offset).scanChars();
            //读完了
        } else if (isEnd()) {
            currentToken = new Token(Assist.END, "", offset);
        } else {
            throw new SQLParsingException(this, Assist.ERROR);
        }
        offset = currentToken.getEndPosition();
    }
```

 

标识符开头：字母、'`'、'_'、'$' 开头的

#### skipIgnoredToken

```java
  private void skipIgnoredToken() {
        //跳过空白符
        offset = new Tokenizer(input, dictionary, offset).skipWhitespace();
        //MySQLLexer是  '/' == getCurrentChar(0) && '*' == getCurrentChar(1) && '!' == getCurrentChar(2)
        //注释开头
        while (isHintBegin()) {
            //算到注释结束
            offset = new Tokenizer(input, dictionary, offset).skipHint();
            //跳过空白符
            offset = new Tokenizer(input, dictionary, offset).skipWhitespace();
        }
      //跳过单行注释、多行注释
      	// '//'或'--'或'/*'开头
        while (isCommentBegin()) {
            offset = new Tokenizer(input, dictionary, offset).skipComment();
            //跳过空白符
            offset = new Tokenizer(input, dictionary, offset).skipWhitespace();
        }
    }
    
```

#### skipHint

处理开头*/结束的东西

```java
public int skipHint() {
    //   HINT_BEGIN_SYMBOL_LENGTH = 3
    return untilCommentAndHintTerminateSign(HINT_BEGIN_SYMBOL_LENGTH);
}

private int untilCommentAndHintTerminateSign(final int beginSymbolLength) {
        int length = beginSymbolLength;
    //非 */
        while (!isMultipleLineCommentEnd(charAt(offset + length), charAt(offset + length + 1))) {
            //如果是EOI = '0x1A'
            if (CharType.isEndOfInput(charAt(offset + length))) {
                throw new UnterminatedCharException("*/");
            }
            length++;
        }
    //offset加中间字符加*/的长度，COMMENT_AND_HINT_END_SYMBOL_LENGTH = 2
        return offset + length + COMMENT_AND_HINT_END_SYMBOL_LENGTH;
    }

    private boolean isMultipleLineCommentEnd(final char ch, final char next) {
        return '*' == ch && '/' == next;
    }
```

#### skipComment

处理注释

```java
public int skipComment() {
    //当前字符
    char current = charAt(offset);
    //下一个
    char next = charAt(offset + 1);
    // 单行注释  '//' 或者'--'
    if (isSingleLineCommentBegin(current, next)) {
        //跳过这行
        return skipSingleLineComment(COMMENT_BEGIN_SYMBOL_LENGTH);
        //一个'#' 
    } else if ('#' == current) {
        //跳过这行
        return skipSingleLineComment(MYSQL_SPECIAL_COMMENT_BEGIN_SYMBOL_LENGTH);
        //跳过多行注释
    } else if (isMultipleLineCommentBegin(current, next)) {
        return skipMultiLineComment();
    }
    return offset;
}


   private boolean isSingleLineCommentBegin(final char ch, final char next) {
        return '/' == ch && '/' == next || '-' == ch && '-' == next;
    }


    private int skipSingleLineComment(final int commentSymbolLength) {
        int length = commentSymbolLength;
        //非 EOI  = '0x1A'或'\n'换行
        while (!CharType.isEndOfInput(charAt(offset + length)) && '\n' != charAt(offset + length)) {
            length++;
        }
        return offset + length + 1;
    }
```

#### scanVariable

处理变量

```java
public Token scanVariable() {
    int length = 1;
    // 第二个也是'@'  就是局部变量@@
    if ('@' == charAt(offset + 1)) {
        length++;
    }
    //变量的符号
    while (isVariableChar(charAt(offset + length))) {
        length++;
    }
    //变量
    return new Token(Literals.VARIABLE, input.substring(offset, offset + length), offset + length);
}

  private boolean isVariableChar(final char ch) {
      //变量标识符或者'.'
        return isIdentifierChar(ch) || '.' == ch;
    }

   private boolean isIdentifierChar(final char ch) {
       //字母、数字、'_'、'$'、'#'
        return CharType.isAlphabet(ch) || CharType.isDigital(ch) || '_' == ch || '$' == ch || '#' == ch;
    }

```

#### scanIdentifier

处理标识符

```java
public Token scanIdentifier() {
    // 第一个字符是'`'
    if ('`' == charAt(offset)) {
        //比如  "`a`" 就长度3
        int length = getLengthUntilTerminatedChar('`');
        return new Token(Literals.IDENTIFIER, input.substring(offset, offset + length), offset + length);
    }
    int length = 0;
    //字母、数字、'_'、'$'、'#'
    while (isIdentifierChar(charAt(offset + length))) {
        length++;
    }
    //取出标识符
    String literals = input.substring(offset, offset + length);
    //Order 或者 Group
    if (isAmbiguousIdentifier(literals)) {
        return new Token(processAmbiguousIdentifier(offset + length, literals), literals, offset + length);
    }
    // 识别关键字，返回类型，不是关键字则使用IDENTIFIER类型
    return new Token(dictionary.findTokenType(literals, Literals.IDENTIFIER), literals, offset + length);
}


 private int getLengthUntilTerminatedChar(final char terminatedChar) {
        int length = 1;
     // 接下来不是 '`'或者连续两个'`'
        while (terminatedChar != charAt(offset + length) || hasEscapeChar(terminatedChar, offset + length)) {
            //一直匹配不到结束，写的sql有问题
            if (offset + length >= input.length()) {
                throw new UnterminatedCharException(terminatedChar);
            }
            //连续两个'`'的，往后再多走一位。走两位
            if (hasEscapeChar(terminatedChar, offset + length)) {
                length++;
            }
            length++;
        }
        return length + 1;
    }

	
    private boolean hasEscapeChar(final char charIdentifier, final int offset) {
        return charIdentifier == charAt(offset) && charIdentifier == charAt(offset + 1);
    }
   


    private TokenType processAmbiguousIdentifier(final int offset, final String literals) {
        int i = 0;
        while (CharType.isWhitespace(charAt(offset + i))) {
            i++;
        }
        if (DefaultKeyword.BY.name().equalsIgnoreCase(String.valueOf(new char[] {charAt(offset + i), charAt(offset + i + 1)}))) {
            //查找关键字类型GROUP或者Order
            return dictionary.findTokenType(literals);
        }
        return Literals.IDENTIFIER;
    }
```

#### scanHexDecimal

处理十六进制

```java
public Token scanHexDecimal() {
    int length = HEX_BEGIN_SYMBOL_LENGTH;
    //负数
    if ('-' == charAt(offset + length)) {
        length++;
    }
    while (isHex(charAt(offset + length))) {
        length++;
    }
    return new Token(Literals.HEX, input.substring(offset, offset + length), offset + length);
}

private boolean isHex(final char ch) {
    return ch >= 'A' && ch <= 'F' || ch >= 'a' && ch <= 'f' || CharType.isDigital(ch);
}
```

#### scanNumber

处理数字

```java
public Token scanNumber() {
    int length = 0;
    //负数
    if ('-' == charAt(offset + length)) {
        length++;
    }
    //数字长度
    length += getDigitalLength(offset + length);
    boolean isFloat = false;
    //小数部分
    if ('.' == charAt(offset + length)) {
        isFloat = true;
        length++;
        length += getDigitalLength(offset + length);
    }
    //科学技术法 'e'或'E'
    if (isScientificNotation(offset + length)) {
        isFloat = true;
        length++;
        if ('+' == charAt(offset + length) || '-' == charAt(offset + length)) {
            length++;
        }
        length += getDigitalLength(offset + length);
    }
    //'f'、'F'、'd'、'D'
    if (isBinaryNumber(offset + length)) {
        isFloat = true;
        length++;
    }
    return new Token(isFloat ? Literals.FLOAT : Literals.INT, input.substring(offset, offset + length), offset + length);
}
```

#### scanSymbol

处理符号包起来的

其中Symbol后面讲述

```java
public Token scanSymbol() {
    int length = 0;
    //跳过符号
    while (CharType.isSymbol(charAt(offset + length))) {
        length++;
    }
    String literals = input.substring(offset, offset + length);
    Symbol symbol;
    //通过字面量查找词法符号.
    while (null == (symbol = Symbol.literalsOf(literals))) {
        literals = input.substring(offset, offset + --length);
    }
    return new Token(symbol, literals, offset + length);
}
```

#### scanChars

处理单引号或双引号扩起来的

```java
public Token scanChars() {
    return scanChars(charAt(offset));
}
//如'avv'、"abb"
private Token scanChars(final char terminatedChar) {
    int length = getLengthUntilTerminatedChar(terminatedChar);
    return new Token(Literals.CHARS, input.substring(offset + 1, offset + length - 1), offset + length);
}
```

### Token

Token有三个参数：

- type(TokenType): INT, FLOAT, HEX, CHARS, IDENTIFIER, VARIABLE
- literals(String): 字面量
- endPosition(int): 字符结束的位置

### Dictionary

字典

  包含一个map，包含每个关键字，Keyword的实现类都是一些枚举常量

```java
 //所有关键字
private final Map<String, Keyword> tokens = new HashMap<>(1024);
```

比如MySQLLexer 实现的时候就是使用MySQLKeyword.values()作为构造参数，构造后填充tokens

```java

    public Dictionary(final Keyword... dialectKeywords) {
        fill(dialectKeywords);
    }
    
    private void fill(final Keyword... dialectKeywords) {
        for (DefaultKeyword each : DefaultKeyword.values()) {
            tokens.put(each.name(), each);
        }
        for (Keyword each : dialectKeywords) {
            tokens.put(each.toString(), each);
        }
    }
}
```

![image](https://user-images.githubusercontent.com/7789698/39878798-d15fb552-54ab-11e8-9111-6bd07a66fcbc.png)

下面这两个方法用来进行词法分析的时候根据字典识别出类型。

```java
TokenType findTokenType(final String literals, final TokenType defaultTokenType) {
    String key = null == literals ? null : literals.toUpperCase();
    return tokens.containsKey(key) ? tokens.get(key) : defaultTokenType;
}

TokenType findTokenType(final String literals) {
    String key = null == literals ? null : literals.toUpperCase();
    if (tokens.containsKey(key)) {
        return tokens.get(key);
    }
    throw new IllegalArgumentException();
}
```


### Symbol

词法符号

包含一个map，包含每个符号，Symbol的都是一些枚举常量

```java
private static Map<String, Symbol> symbols = new HashMap<>(128);
```



获取符号具体类型

```java
public static Symbol literalsOf(final String literals) {
    return symbols.get(literals);
}
```

