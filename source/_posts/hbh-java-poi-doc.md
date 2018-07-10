---
title: 文书生成模块文档
date: 2018-03-19 20:11:12
tags:
    - Apache POI
categories: 网页开发
---
这是一个使用 Apache POI 开源库进行文书模板生成的模块。
<!-- more -->
# Word 模板键设置

需要修改 Word 文档，使其成为一个模板，才能使用这个包。
模板中需要动态添加的字段或表格，使用 `${key}` 代替，。
其中，`key` 可以替换为自己的字段，`key` 称为“**模板键**”。
为了提高开发效率，对模板键的设置做如下约定：

- 单字段（`DocElementField`）使用 `${EL_}` 开头的模板键（如`${EL_recorder}`）
- 表格（`DocTableField`）使用 `${TB_}` 开头的模板键（如`${TB_indicators}`）

> 这样做的目的是，可以让一个人去做 Word 模板，另一个人去针对每个 Word 文书制作生成器（`DocGenerator`）。

示例：

{% asset_img doc-template-key.png %}

> 为了避免格式错误，应充分使用*制表位*、*边距*、*对齐方式*
等 Word 格式设置代替原文档中的大量空格。

# 类的使用方法

## 所有类一览 

下面是模块中的所有类：

- `IDocField` 接口：表示文档中的字段。
    - `DocElementField` 类：单子段。
    - `DocTableField` 类：表格。
- `DocFieldType` 枚举：表示文档字段的类型。
- `DocTableFieldHeader` 类：表示表头。
- `DocTableFieldCell` 类：表示表单元格。
- `DocGenerator` 类：文档渲染器。

类的详细信息可参考 [JavaDoc][javadoc]。

## 基本使用

创建 `DocGenerator` 对象，设置其以下属性：

- `docPath`：源文档路径。
- `docSavePath`：生成文档的保存路径。
- `fieldMap`：字段映射，为键值对形式。键即为文档模板键，值为 `IDocField` 对象。

然后调用 `loadDocument()` 方法加载文档，使用 `replaceInDoc()` 方法生成文档。

完成示例如下：

``` java
package com.zkty.hbh;

import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import org.apache.poi.xwpf.usermodel.UnderlinePatterns;

public class App 
{
    public static void main( String[] args )
    {
    		DocGenerator generator = new DocGenerator();
    		generator.docPath = "/formCaseSource.docx";
    		generator.docSavePath = "/formCaseSource-output.docx";
    		generator.fieldMap.put("${EL_reportYear}", new DocElementField("2017"));
    		generator.fieldMap.put("${EL_reportMonth}", new DocElementField("3"));
    		generator.fieldMap.put("${EL_reportDay}", new DocElementField("19"));
    		generator.fieldMap.put("${EL_reportHour}", new DocElementField("19"));
    		
    		// 创建表格参数
    		List<DocTableFieldHeader> headerMap = new ArrayList<DocTableFieldHeader>();
    		headerMap.add(new DocTableFieldHeader("test1", "测试1"));
    		headerMap.add(new DocTableFieldHeader("test2", "测试2"));
    		headerMap.add(new DocTableFieldHeader("test3", "测试3"));
    		DocTableFieldHeader header4 = new DocTableFieldHeader("test4", "测试4");
    		header4.width = 2000;
    		headerMap.add(header4);
    		headerMap.add(new DocTableFieldHeader("test5", "测试5"));
    		List<Map<String, DocTableFieldCell>> contentMap = new ArrayList<Map<String, DocTableFieldCell>>();
    		for (int i = 0; i < 10; i++) {
    			Map<String, DocTableFieldCell> content = new HashMap<String, DocTableFieldCell>();
    			for (int j = 0; j < 5; j++) {
    				content.put("test" + (j + 1), new DocTableFieldCell(("测试内容" + (i + 1)) + (j + 1)));
			    }
    			contentMap.add(content);
		    }
    		generator.fieldMap.put("${TB_demo}", new DocTableField(headerMap, contentMap));
    		try {
				generator.loadDocument();
				generator.showDocument();
				generator.replaceInDoc();
				generator.showDocument();
				generator.saveDocument();
			} catch (IOException e) {
				System.out.println("Open document filed!");
				e.printStackTrace();
			}
    }
}

```

## 其他使用方法

如果当前定义的两个字段类渲染出的样式无法满足需求，可以从其派生，
派生后添加自己的属性，然后重载 `setStyle()` 函数，自定义样式。

-----------

生成效果图示

{% asset_img render-result.png %}

[javadoc]:https://hpdell.github.io/hbh-java-poi-render/