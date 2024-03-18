---
title: Guide to Navigating PDF Internals
draft: false
authors:
  - admin
date: 2024-03-18
tags:
  - PDF
  - QPDF
  - JQ
  - CLI
---


I'm working on a little tool for learning a new language and in that project I need to read contents from a PDF. PDFs are more than just a collection of text formatted in a clean way. The structure of a PDF is that of a dictionary. In this post you'll learn with me the basics of the PDFs internal structure as well as learn to use **QPDF**.

## QPDF

A tool I found for navigating a PDF. It's a cli tool used to explore and even construct a PDF. It's simple and straight forward to get started with. We'll also pair QPDF with JQ. JQ is another cli tool used to slice and dice JSON.

[QPDF Documentation](https://qpdf.readthedocs.io/en/stable/overview.html)

In this post we'll use QPDF to view and learn the structure of a PDF file. As you'll see PDFs are not markdown tables. It's its own language.


## Key Questions Answered in This Guide

- What is the basic structure of a PDF file?
- How does the PDF file structure differ from simpler document formats like Markdown?
- How can QPDF be used to explore the internal structure of a PDF?
- What are the basic commands to get started with QPDF for PDF exploration?
- How can the content of a PDF be dumped into a JSON format using QPDF?
- What are the benefits of converting PDF content to JSON?
- What are the main components of a PDF's base structure (File Header, Body, Cross-Reference Table, Trailer Dictionary)?
- How can specific objects or information be extracted from a PDF using QPDF?
- Besides QPDF, what other tools can assist in navigating and inspecting PDF internals?
- How can web-based PDF internal viewers complement the functionality of QPDF?


Lets run through some examples with QPDF and we'll explore the PDF internals after.

## JSON

JSON is one of the most if not the most used data structure out. Fortunately for us, QPDF allows up to dump the output of a PDF into JSON

JSON to file
```bash
qpdf --json in.pdf out.json
```

JSON to stdout
```bash
qpdf --json in.pdf
```

Filtering the JSON output
```bash
qpdf --json --json-object=qpdf in.pdf
```

Filtering with JQ and less, where the `-C` in jq will maintain the colors.
```bash
qpdf --json in.pdf | jq -C . | less
```


{{% callout note %}}
Often, JSON object will be large. To explore the JSON object you could use JQ and less or copy the out from --json to [JSON Formatter](https://jsonformatter.org/json-viewer) to give you a more controlled view.
{{% /callout %}}

## Base Structure

A PDF is a collection of objects jammed into a file. A PDF reader knows how to view the PDF and render it to the screen. Before we get into how the PDF is read lets look at the Objects within a PDF


- File Header
- Body
- Cross Reference Table
- Trailer Dictionary


### File Header

The File Header contains the version of the PDF file and a collection of bytes.

```js
%PDF - <version> 
%<bytes>
```

Here's how to extract the PDF version using QPDF

```bash
qpdf --json in.pdf | jq .qpdf[]| select(.pdfversion)
```

{{% callout note %}}
More details about the File Headers can be found in section `7.5.2` of the [PDF version 1.7 Reference](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf)
{{% /callout %}}

### Body

The body contains all the content inside the PDF. That's pages, graphical content and other objects needed for rendering the PDF. Each piece of content is an object inside the PDF. We'll explore more the object in later content but for now we'll focus on navigating the base structure.  

Most of the objects are structured in a similar way with varying keys and values. Like a Map or Dictionary you have Keys `/Type`, `/Count` `/etc..` with following values. The keys will either have the value of the key or a **Reference** to another object. e.g `[2 0 R]`. 


```js
1 0 obj
<<
/Kids [2 0 R]
/Count 1
/Type /Pages
>>
endobj
```

`1 0 obj` is the reference to this obj. Where `[2 0 R]` is the location of the Kids. You'd think about accessing to the kids object like `1 0 obj` -> `/Kids` -> `2 0 obj`.


Using QPDF to pull object 25 from in.pdf

```bash
qpdf --json --show-object=25 in.pdf
```

{{% callout note %}}
More details about the Body can be found in section `7.5.3` of the [PDF version 1.7 Reference](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf)
{{% /callout %}}

### Cross-Reference Table

This table contains the byte offset location of each object in the file. Its setup like this for random access so the file will not need to be read to access any object. The location of this table can be found in the Trailer where you find other objects like the Catalog.

```js
xref
0 7 # specify the range of subsections
0000000000 65535 f # f = entry thats free
0000000015 00000 n # n = entry thats in use
0000000066 00000 n 
0000000126 00000 n 
0000000265 00000 n 
0000000350 00000 n 
0000000434 00000 n
```

{{% callout note %}}
More details about the Cross-Reference Table can be found in section `7.5.4` of the [PDF version 1.7 Reference](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf)
{{% /callout %}}

### Trailer

The Trailer object serves as the map of the PDF. Showing where the PDF starts for the xref table and other special objects like the catolog/root.
`startxref` followed by `454` indicates byte offset of the xref object.

```js
trailer
<<
    /Size 7
    /Root 1 0 R # Document Catalog
    /Info 6 0 R # Metadata about the document
>>
startxref
454
%%EOF
```

{{% callout note %}}
More details about the Trailer can be found in section `7.5.5` of the [PDF version 1.7 Reference](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf)
{{% /callout %}}

# Conclusion

We have laid the foundation for understanding PDF internals. After reading this post and experimenting with QPDF, you're ready to delve deeper into the Reference guide.

PDFs are complex, with a unique structure that differs significantly from simpler formats like Markdown. By utilizing CLI tools such as QPDF, we can efficiently process PDFs and even automate the extraction of meaningful content through scripting. The combination of tools like QPDF and JQ demonstrates the power of filtering out the noise to get to the information we need.

Remember, as with any learning process, grasping the basics of a topic paves the way for exploring its depths. In our next post, we'll dive into extracting text from PDFs and take a closer look at what Object Streams are.

## Resource
- [PDF version 1.7 Reference](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/PDF32000_2008.pdf)
- [Web based PDF internal viewer](https://pdfux.com/inspect-pdf/)
- [JQ Documentation](https://jqlang.github.io/jq/manual/)

