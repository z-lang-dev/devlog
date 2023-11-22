## PDF生成

先读取`chapters.txt`，获得所有.md文件的路径，再

```powershell
(Get-Content .\chapters.txt -Raw) -replace '\n',' '
```

```bash
pandoc --pdf-engine=wkhtmltopdf -o book.pdf <加上上面的文件列表>
```
