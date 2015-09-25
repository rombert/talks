SLIDES=intro.md
TEMPLATE=adaptTo2015_PresentationTemplate_trimmed.odp
OUTPUT=How_Do_I_Test_My_Sling_Application-Robert_Munteanu.odp
ODPDOWN=../tools/odpdown
.PHONY=clean

$(OUTPUT): $(SLIDES) $(TEMPLATE) $(ASSETS) assets/scaled/*
	$(ODPDOWN)/odpdown --break-master=Topic_5f_v2 --content-master=Content_5f_v2 \
        --code-font-name='Source Code Pro Light' $(SLIDES) $(TEMPLATE) $(OUTPUT)

clean:
	rm -f $(OUTPUT)
