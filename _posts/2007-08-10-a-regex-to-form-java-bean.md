---
layout: default
title: A regex to form java bean
---
First declare sets of member variables like below:

    private int id;

    private String name;

    private int industry;

    private String address;


Then use the Find and Replace function of your text editor.

Find:

    \tprivate (\w+) (\w+);


Replace to:

    public \1 get\2\(\)\r\n{\r\n\treturn this\.\2;\r\n}\r\nprivate void set\2\(\1 \2\)\r\n{\r\n\tthis\.\2 = \2;\r\n}


And paste the result back to the .java file.If your favourite editor is UltraEdit32 (as me), remember to check Unix style Regular Expressions in the configuration.

Just for memorization.

