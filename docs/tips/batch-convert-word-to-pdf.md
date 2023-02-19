---
title: Windows 下批量转换 word 文档为 pdf 的方法
description: Windows 下如何利用 PowerShell 来批量将 Word 文档转为 pdf 格式的文档
tags: ["word", "pdf", "powershell"]
---

# Windows 下批量转换 word 文档为 pdf 的方法

方法为利用PowerShell的插件功能

首先从官方网站下载 `ConvertWordDocumentToPDS.psm1` 文件，假定下载到 `d:\tool` 下，
然后打开powershell，执行下面的命令，将插件导入

```
Import-Module D:\tool\ConvertPowerPointToWordDocument.psm1
```

然后就可以执行下面的命令了

```
ConvertTo-OSCPDF –Path D:\Word
```

其中 `D:\word` 就是你要转换word文件的目录，脚本会把改目录下的所有word文档转为同名的pdf文件，并存放在当前目录下