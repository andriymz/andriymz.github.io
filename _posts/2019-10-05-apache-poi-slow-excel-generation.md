---
title:  "Solving Slow Excel Generation using Apache POI"
excerpt: "Comparing standard and memory optimized Excel generations using Apache POI library"
category: [misc]
classes: wide
author_profile: true
---

[Apache POI](https://poi.apache.org/) is a _Java APIs for manipulating various file formats based upon the Office Open XML standards (OOXML) and Microsoft's OLE 2 Compound Document format (OLE2). In short, you can read and write MS Excel files using Java. In addition, you can read and write MS Word and MS PowerPoint files using Java._

If you try to generate an Excel using the [XSSFWorkbook](http://poi.apache.org/apidocs/dev/org/apache/poi/xssf/usermodel/XSSFWorkbook.html) class you may experience extreme slowness, because apparently all the added to the workbook cells are kept in-memory until the Excel is saved.

You can easily find a lot of threads (e.g [this](https://stackoverflow.com/questions/2498536/poi-performance)) about this problem and the solution is using the [SXSSFWorkbook](https://poi.apache.org/apidocs/dev/org/apache/poi/xssf/streaming/SXSSFWorkbook.html) class instead. Both implementations implement the [Workbook](http://poi.apache.org/apidocs/dev/org/apache/poi/ss/usermodel/Workbook.html) interface.

SXSSFWorkbook is a _Streaming version of XSSFWorkbook implementing the "BigGridDemo" strategy. This allows to write very large files without running out of memory as only a configurable portion of the rows are kept in-memory at any one time._

When instantiating the SXSSFWorkbook class, you provide a window size parameter which represents the amount of rows which will be kept in-memory. All the exceeding rows will be flushed to disk.

Although the solution is simple, by simply switching the workbook class, I wanted to analyze how bad the XSSFWorkbook's performance is and how the SXSSFWorkbook improves it. For that I registered the time both classes take to generate an Excel with 20 columns and the following number of rows: 10000, 50000, 100000, 200000, 500000, 1000000 and 1048575 (maximum supported row number).

After confirming that the SXSSFWorkbook has better performance, I also registered the Excel generation time with the following in-memory row windows: 100, 200, 500, 1000, 5000, 10000 and 50000.


## Results

To test the performance I ran the code provided in the [Code](#code) section, first by providing no argument to run the in-memory based Excel generation, and then by passing the `stream` argument to run the window based Excel generation. The main difference between the executions was the maximum Heap size, which was `-Xmx4096m` and `-Xmx2046m`, respectively.

As you can see in the ouput below from the in-memory based implementation, the execution time increases linearly and worse than that, is the insufficient Heap size.

```
Building workbook in normal mode
Elapsed 3671 millis for 10000 rows
Elapsed 15416 millis for 50000 rows
Elapsed 28284 millis for 100000 rows
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
        at java.util.Arrays.copyOfRange(Arrays.java:3664)
        at java.lang.String.<init>(String.java:207)
        at java.lang.StringBuilder.toString(StringBuilder.java:407)
        at org.apache.poi.ss.util.CellReference.convertNumToColString(CellReference.java:474)
        at org.apache.poi.ss.util.CellReference.appendCellReference(CellReference.java:530)
        at org.apache.poi.ss.util.CellReference.formatAsString(CellReference.java:495)
        at org.apache.poi.xssf.usermodel.XSSFCell.setCellNum(XSSFCell.java:878)
        at org.apache.poi.xssf.usermodel.XSSFRow.createCell(XSSFRow.java:223)
        at org.apache.poi.xssf.usermodel.XSSFRow.createCell(XSSFRow.java:198)
        at org.apache.poi.xssf.usermodel.XSSFRow.createCell(XSSFRow.java:45)
```

On the other hand, the window based implementation is capable of running in a considerably reduced amount of time. As you can see, there is no considerable difference between the different row window sizes, except the larger window of 50000 rows. With the latter window the application shows higher Heap and CPU usage.

Ignore the first two executions, which were used for the JVM warm-up.

```
Building workbook in normal mode
Elapsed 82 millis for 1 rows
Elapsed 11 millis for 2 rows
Elapsed 14 millis for 3 rows
...
Elapsed 19 millis for 98 rows
Elapsed 19 millis for 99 rows
Elapsed 19 millis for 100 rows

Building workbook in stream mode with 1000 row window
Elapsed 165 millis for 1 rows
Elapsed 2 millis for 2 rows
Elapsed 2 millis for 3 rows
...
Elapsed 2 millis for 98 rows
Elapsed 2 millis for 99 rows
Elapsed 2 millis for 100 rows

Building workbook in stream mode with 100 row window
Elapsed 516 millis for 10000 rows
Elapsed 2194 millis for 50000 rows
Elapsed 4431 millis for 100000 rows
Elapsed 8665 millis for 200000 rows
Elapsed 23958 millis for 500000 rows
Elapsed 44459 millis for 1000000 rows
Elapsed 48679 millis for 1048575 rows

Building workbook in stream mode with 200 row window
Elapsed 459 millis for 10000 rows
Elapsed 2294 millis for 50000 rows
Elapsed 4606 millis for 100000 rows
Elapsed 9404 millis for 200000 rows
Elapsed 23654 millis for 500000 rows
Elapsed 44025 millis for 1000000 rows
Elapsed 46341 millis for 1048575 rows

Building workbook in stream mode with 500 row window
Elapsed 430 millis for 10000 rows
Elapsed 2183 millis for 50000 rows
Elapsed 4682 millis for 100000 rows
Elapsed 8966 millis for 200000 rows
Elapsed 22455 millis for 500000 rows
Elapsed 44293 millis for 1000000 rows
Elapsed 53461 millis for 1048575 rows

Building workbook in stream mode with 1000 row window
Elapsed 445 millis for 10000 rows
Elapsed 2433 millis for 50000 rows
Elapsed 4481 millis for 100000 rows
Elapsed 9141 millis for 200000 rows
Elapsed 22924 millis for 500000 rows
Elapsed 44966 millis for 1000000 rows
Elapsed 50804 millis for 1048575 rows

Building workbook in stream mode with 5000 row window
Elapsed 292 millis for 10000 rows
Elapsed 2126 millis for 50000 rows
Elapsed 4433 millis for 100000 rows
Elapsed 8866 millis for 200000 rows
Elapsed 22163 millis for 500000 rows
Elapsed 43857 millis for 1000000 rows
Elapsed 50241 millis for 1048575 rows

Building workbook in stream mode with 10000 row window
Elapsed 107 millis for 10000 rows
Elapsed 2317 millis for 50000 rows
Elapsed 4308 millis for 100000 rows
Elapsed 8898 millis for 200000 rows
Elapsed 22792 millis for 500000 rows
Elapsed 44153 millis for 1000000 rows
Elapsed 50700 millis for 1048575 rows

Building workbook in stream mode with 50000 row window
Elapsed 98 millis for 10000 rows
Elapsed 604 millis for 50000 rows
Elapsed 3426 millis for 100000 rows
Elapsed 8415 millis for 200000 rows
Elapsed 22838 millis for 500000 rows
Elapsed 185916 millis for 1000000 rows
Elapsed 235439 millis for 1048575 rows
```

In terms of the used memory, [Java VisualVM](https://visualvm.github.io) shows that the maximum Heap size of 2GB was enough to generate all the Excel workbooks. Keep in mind that these tests represent the worst case scenario, where before constructing the workbook we already have a grid (array of arrays) of allocated on the Heap objects, where each object contains the cell value. An optimized implementation using iterators (if possible), that fetches each row from the source only when required to populate the workbook, will have a very low memory footprint. This, combined with the sliding-window based workbook implementation will result in a fast and light Excel generation.

{% include figure image_path="/assets/images/poi/jvisualvm.png" %}

## Code

```scala 
import java.io.FileOutputStream
import java.util.UUID

import org.apache.poi.ss.usermodel.Workbook
import org.apache.poi.xssf.streaming.SXSSFWorkbook
import org.apache.poi.xssf.usermodel.XSSFWorkbook

object TestExcelGeneration {

  private val rowNumberValues = Seq(10000, 50000, 100000, 200000, 500000, 1000000, 1048575)
  private val streamWindowValues = Seq(100, 200, 500, 1000, 5000, 10000, 50000)

  val currentMillis = System.currentTimeMillis()

  def main(args: Array[String]): Unit = {
    val streamMode = !args.isEmpty && args(0).toLowerCase().startsWith("stream")

    // JIT Warm-up
    buildWorkbook(1 to 100)
    buildWorkbook(1 to 100, streamMode = true)

    if(streamMode) {
      streamWindowValues.foreach { streamWindow =>
        buildWorkbook(rowNumberValues, streamMode = true, streamWindow)
      }
    }
    else {
      buildWorkbook(rowNumberValues)
    }
  }

  def buildWorkbook(rowNumberValues: Seq[Int], streamMode: Boolean = false, streamWindow: Int = 1000): Unit = {
    println(s"\nBuilding workbook in ${if(streamMode) s"stream mode with $streamWindow row window" else "normal mode"}")

    rowNumberValues.foreach { numRows =>
      val content: Seq[Seq[Any]] = (0 until numRows).map { row =>
        Seq(
          row, Math.random() * 1000000, new java.util.Date(currentMillis), UUID.randomUUID(), UUID.randomUUID(),
          UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(),
          UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(),
          UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID()
        )
      }

      val workbook = if(streamMode) new SXSSFWorkbook(streamWindow) else new XSSFWorkbook()

      val thenMillis = System.currentTimeMillis()
      addSheet(workbook, s"name_$numRows", content)
      val nowMillis = System.currentTimeMillis()
      val elapsedMillis = nowMillis - thenMillis

      workbook.write(new FileOutputStream(s"/tmp/excel/excel_$numRows.xlsx"))

      println(s"Elapsed $elapsedMillis millis for $numRows rows")
    }
  }

  def addSheet(workbook: Workbook, name: String, content: Seq[Seq[Any]]) = {
    val wkbSheet = workbook.createSheet(name)
    val totalRows = content.size
    val totalCells = content.head.size

    (0 until totalRows).foreach { rowId =>
      val row = wkbSheet.createRow(rowId)

      (0 until totalCells).foreach { cellId =>
        val cell = row.createCell(cellId)
        content(rowId)(cellId) match {
          case v if v == null => cell.setBlank()
          case v: Int => cell.setCellValue(v)
          case v: Long => cell.setCellValue(v)
          case v: Double => cell.setCellValue(v)
          case v: Float => cell.setCellValue(v)
          case v: Boolean => cell.setCellValue(v)
          case v: String => cell.setCellValue(v)
          case v => cell.setCellValue(v.toString) // For other types just set their String representation
        }
      }
    }
  }
}
```
