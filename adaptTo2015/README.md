Authoring remarks
==

Adding notes
--

The notes can be added using the 

	~~~Comment
	YOUR_COMMENT_HERE
	~~~

syntax. For now, they must be placed immediately under the heading element to appear in the final output.

Manual tweaking after generation
--

After generating the slides using `make`, make sure to edit the _unedited.odp file and

* bold out text in code samples where needed
* resize/aling images ( see [odpdown bug #30](https://github.com/thorstenb/odpdown/issues/30) )
  * 0.00 x 0.96 position, 10.00 x 4.34 size