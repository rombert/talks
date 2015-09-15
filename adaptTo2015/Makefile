SLIDES=intro.md
TEMPLATE=adaptTo2015_PresentationTemplate_trimmed.odp
OUTPUT=How_Do_I_Test_My_Sling_Application-Robert_Munteanu.odp
ODPDOWN=../tools/odpdown

$(OUTPUT): $(SLIDES) $(TEMPLATE) assets/scaled/automatedtestingpyramid.png assets/scaled/comparison.gif assets/original/nosql-unit-test-execution-time.png
	export PYTHONPATH=$(ODPDOWN)
	$(ODPDOWN)/odpdown --break-master=Topic_5f_v2 --content-master=Content_5f_v2 $(SLIDES) $(TEMPLATE) $(OUTPUT)

