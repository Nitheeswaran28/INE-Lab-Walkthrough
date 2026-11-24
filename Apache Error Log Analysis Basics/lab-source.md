# Apache Error Log Analysis Basics source notes

INE/AttackDefense walkthrough 140 supplies the commands and expected results. Its screenshots show the reported username total, examples of invalid URI evidence, and excerpts from the two exports.

- The missing-file filter reports 229,112 records in both Q1 and Q5. This is a row count, with repeated paths included.
- The username extraction reports 75 distinct strings. The screenshot displays part of the username export.
- The Invalid URI filter reports 15 matching entries. The manual lists eight target filenames.
- Both export commands write to the same file. The PDF shows a preview of the path export.

The original dataset endpoint returned HTTP 404. The original error.log and full export contents could not be independently verified. Counts are attributed to the PDF.

[Original Apache error log URL](http://www.almhuette-raith.at/apache-log/error.log)
