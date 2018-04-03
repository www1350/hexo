---
title: 解决emoji存储
date: 2018-04-03 22:39:29
tags:
categories:
---

1.utf8mb4的最低mysql版本支持版本为5.5.3+，若不是，请升级到较新版本。

2.修改mysql配置文件my.cnf（windows为my.ini）
my.cnf一般在etc/mysql/my.cnf位置。找到后请在以下三部分里添加如下内容：

```
[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4

[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4'

```

在mysql中执行：

```
set character_set_client = utf8mb4;
set character_set_connection = utf8mb4;
set character_set_database = utf8mb4;
set character_set_results = utf8mb4;
set character_set_server = utf8mb4;
```

重启mysql
Linux:`service mysql restart`

3.修改database、table和column字符集。参考以下语句：

```
ALTER DATABASE database_name CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
ALTER TABLE table_name CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
ALTER TABLE table_name MODIFY COLUMN column_name VARCHAR(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci ;
```

查看是否修改成功
`SHOW VARIABLES WHERE Variable_name LIKE 'character\_set\_%' OR Variable_name LIKE 'collation%';  
`

4.如果你用的是java服务器，升级或确保你的mysql connector版本高于5.1.13，否则仍然无法使用utf8mb4

5.jdbc的url必须&characterEncoding=utf8

6.备份数据库的时候
`mysqldump -uroot -p --default-character-set=utf8mb4 --hex-blob databasename > databasename.sql`

**注：在navicat里面会看到乱码**
可选：

7.解决不兼容问题

```
public class EmojiUtil {

    public static String[] ios5emoji ;
    public static String[] ios4emoji ;
    public static String[] androidnullemoji ;
    public static String[] adsbuniemoji;

    public static void initios5emoji(String[] i5emj,String[] i4emj,String[] adnullemoji,String[] adsbemoji){
        ios5emoji = i5emj;
        ios4emoji = i4emj;
        androidnullemoji = adnullemoji;
        adsbuniemoji = adsbemoji;
    }

    //在ios上将ios5转换为ios4编码
    public static String transToIOS4emoji(String src) {
        return StringUtils.replaceEach(src, ios5emoji, ios4emoji);
    }
    //在ios上将ios4转换为ios5编码
    public static String transToIOS5emoji(String src) {
        return StringUtils.replaceEach(src, ios4emoji, ios5emoji);
    }
    //在android上将ios5的表情符替换为空
    public static String transToAndroidemojiNull(String src) {
        return StringUtils.replaceEach(src, ios5emoji, androidnullemoji);
    }

    //在android上将ios5的表情符替换为SBUNICODE
    public static String transToAndroidemojiSB(String src) {
        return StringUtils.replaceEach(src, ios5emoji, adsbuniemoji);
    }

    //在android上将SBUNICODE的表情符替换为ios5
    public static String transSBToIOS5emoji(String src) {
        return StringUtils.replaceEach(src, adsbuniemoji, ios5emoji);
    }

    //eg. param: 0xF0 0x9F 0x8F 0x80
    public static String hexstr2String(String hexstr) throws UnsupportedEncodingException{
        byte[] b = hexstr2bytes(hexstr);
        return new String(b, "UTF-8");
    }

    //eg. param: E018
    public static String sbunicode2utfString(String sbhexstr) throws UnsupportedEncodingException{
        byte[] b = sbunicode2utfbytes(sbhexstr);
        return new String(b, "UTF-8");
    }

    //eg. param: 0xF0 0x9F 0x8F 0x80
    public static byte[] hexstr2bytes(String hexstr){
        String[] hexstrs = hexstr.split(" ");
        byte[] b = new byte[hexstrs.length];

        for(int i=0;i<hexstrs.length;i++){
            b[i] = hexStringToByte(hexstrs[i].substring(2))[0];
        }
        return b;
    }

    //eg. param: E018
    public static byte[] sbunicode2utfbytes(String sbhexstr) throws UnsupportedEncodingException{
        int inthex = Integer.parseInt(sbhexstr, 16);
        char[] schar = {(char)inthex};
        byte[] b = (new String(schar)).getBytes("UTF-8");
        return b;
    }

    public static byte[] hexStringToByte(String hex) {
        int len = (hex.length() / 2);
        byte[] result = new byte[len];
        char[] achar = hex.toCharArray();
        for (int i = 0; i < len; i++) {
            int pos = i * 2;
            result[i] = (byte) (toByte(achar[pos]) << 4 | toByte(achar[pos + 1]));
        }
        return result;
    }


    private static byte toByte(char c) {
        byte b = (byte) "0123456789ABCDEF".indexOf(c);
        return b;
    }


    /**
     * 将str中的emoji表情转为byte数组
     * 
     * @param str
     * @return
     */
    public static String resolveToByteFromEmoji(String str) {
        Pattern pattern = Pattern
                .compile("[^(\u2E80-\u9FFF\\w\\s`~!@#\\$%\\^&\\*\\(\\)_+-？（）——=\\[\\]{}\\|;。，、《》”：；“！……’:'\"<,>\\.?/\\\\*)]");
        Matcher matcher = pattern.matcher(str);
        StringBuffer sb2 = new StringBuffer();
        while (matcher.find()) {
            matcher.appendReplacement(sb2, resolveToByte(matcher.group(0)));
        }
        matcher.appendTail(sb2);
        return sb2.toString();
    }

    /**
     * 将str中的byte数组类型的emoji表情转为正常显示的emoji表情
     * 
     * @param str
     * @return
     */
    public static String resolveToEmojiFromByte(String str) {
        Pattern pattern2 = Pattern.compile("<:([[-]\\d*[,]]+):>");
        Matcher matcher2 = pattern2.matcher(str);
        StringBuffer sb3 = new StringBuffer();
        while (matcher2.find()) {
            matcher2.appendReplacement(sb3, resolveToEmoji(matcher2.group(0)));
        }
        matcher2.appendTail(sb3);
        return sb3.toString();
    }

    private static String resolveToByte(String str) {
        byte[] b = str.getBytes();
        StringBuffer sb = new StringBuffer();
        sb.append("<:");
        for (int i = 0; i < b.length; i++) {
            if (i < b.length - 1) {
                sb.append(Byte.valueOf(b[i]).toString() + ",");
            } else {
                sb.append(Byte.valueOf(b[i]).toString());
            }
        }
        sb.append(":>");
        return sb.toString();
    }

    private static String resolveToEmoji(String str) {
        str = str.replaceAll("<:", "").replaceAll(":>", "");
        String[] s = str.split(",");
        byte[] b = new byte[s.length];
        for (int i = 0; i < s.length; i++) {
            b[i] = Byte.valueOf(s[i]);
        }
        return new String(b);
    }

    public static void main(String[] args) throws UnsupportedEncodingException {
        // TODO Auto-generated method stub
        byte[] b1 = {-30,-102,-67}; //ios5 //0xE2 0x9A 0xBD     
        byte[] b2 = {-18,-128,-104}; //ios4 //"E018"

        //-------------------------------------

        byte[] b3 = {-16,-97,-113,-128};    //0xF0 0x9F 0x8F 0x80       
        byte[] b4 = {-18,-112,-86};         //E42A  


        ios5emoji = new String[]{new String(b1,"utf-8"),new String(b3,"utf-8")};
        ios4emoji = new String[]{new String(b2,"utf-8"),new String(b4,"utf-8")};    





        //测试字符串
        byte[] testbytes = {105,111,115,-30,-102,-67,32,36,-18,-128,-104,32,36,-16,-97,-113,-128,32,36,-18,-112,-86};
        String tmpstr = new String(testbytes,"utf-8");
        System.out.println(tmpstr);


        //转成ios4的表情
        String ios4str = transToIOS5emoji(tmpstr);
        byte[] tmp = ios4str.getBytes();
        //System.out.print(new String(tmp,"utf-8"));        
        for(byte b:tmp){
            System.out.print(b);
            System.out.print(" ");
        }
    }

}
```

> 另：
> 如果你不想重启数据库，可以这样做：

1.表必须是utf8mb4

2.连接池
      `  <property name="connectionInitSqls" value="set names utf8mb4;"/>`

或者每次插入前执行
`set names utf8mb4`