---
layout: post
title:  "Oil and Gas Data Challenges"
date:   2019-07-22 12:00:00
categories: data webscraping
---


My second project at Metis involved some of my previous experience in the oil and gas industry. My goal was to describe and predict the completion uplift of various completion designs, taking into account variables such as proppant loading, fluid loading, proppant types and additional chemicals used in the frac such as clay controls and cross linkers. I had expected to find a very strong trend between proppant/fluid loading and completion uplift, but ended up having great difficulties establishing any trends in my dataset. 

I’d like to use this post to discuss how I went about collecting my data and why that data did not produce the answers I expected. I do not believe my thesis was fundamentally incorrect, but rather the quality of publicly reported completion data was insufficient for analysis that requires a high degree of precision in both specifying the question and in actual data quality.

## Collecting the Data

My data collection efforts were focused around two sources. The [FracFocus Chemical Disclosure Registry](https://fracfocus.org/) contains a listing of chemicals used in completions searchable by API Number, a unique identifier for domestic oil and gas wells. The Texas Railroad Commission (RRC) maintains records of completion filings on form W-2s, with PDF forms for specific wells available via an [online query](http://webapps.rrc.texas.gov/CMPL/publicHomeAction.do). These filings have information regarding initial (24-hour) production rates along with lateral length of the completed wells. 

### FracFocus Dataset

The FracFocus dataset is available in both csv and SQL database format and is easy to onboard into Python. However, consistency of naming within the dataset became an issue – there were over 18,000 unique chemicals with 7,500+ different stated uses for the chemicals. Operators would often use different nomenclature for the same chemicals such as calling “100 mesh sand” “100-mesh” or something similar. Operators would also report different levels of detail, some describing proppant as just sand and others going into mesh detail and sometimes geographic origin. I employed regex searches to find common terms and to simplify ingredient names and purposes where possible. Here is a quick example of the lookup dictionary and applicable function:

{% highlight python %}
from collections import OrderedDict

prop_mesh_regex = OrderedDict([
    ('100-mesh',[r'(?i)100M',r'(?i)100 M']),
    ('12/20',[r'12/20',r'12-20']),
    ('16/30',[r'16/30',r'16-30']),
    ('20/40',[r'20/40',r'20-40']),
    ('30/50',[r'30/50',r'30-50']),
    ('40/70',[r'40/70',r'40-70']),
    ('40/140',[r'40/140',r'40-140']),
    ('other',[r'(?i)sand',r'(?i)crc',r'(?i)carbo',r'(?i)ceramic',r'(?i)rcs',r'(?i)nws',r'(?i)silica',r'(?i)prop'])
])

def ranked_lookup(string,regex_dict):
    '''
    Accepts a string and dictionary. Returns the first dictionary key where one regex value is found in the string,
    so the order of the dictionary matters (ordered dictionary recommended).
    '''
    try:
        for key in regex_dict.keys():
            for regex in regex_dict[key]:
                if re.search(regex,string):
                    return key        
        return None
    except TypeError:
        return None
{% endhighlight %}


### RRC Completion Reports

RRC completion reports posed a different challenge for data collection. First, completion report IDs needed to be queried by API number. These report IDs were then used to request PDF reports corresponding to each ID. Finally, PDFs had to be parsed for specific data such as IP rates, measured depth and artificial lift status at test time. 

I used the location of textboxes on each PDF page to identify which pieces of text to scrape from each report. I used the function below to score textboxes based on proximity to expected textbox locations for each point of data:

{% highlight python %}
import pdfminer
from pdfminer.pdfparser import PDFParser, PDFDocument
from pdfminer.pdfinterp import PDFResourceManager, PDFPageInterpreter
from pdfminer.converter import PDFPageAggregator
from pdfminer.layout import LAParams, LTTextBox, LTTextLine

#lookup table maps pieces of information to the textbox location on completion report pdfs
#see box_print() at the bottom of the notebook for a print of all textboxes 
#tuple is (page num,bbox[0],bbox[1],bbox[2],bbox[3])
lookup_table = {
    (0,18,672,212,746) : 'api_num',
    (0,18,417,209,561) : 'file_purpose', 
    (0,18,132,383,390) : 'well_info',
    (0,308,482,505,529) : 'comp_date',
    (1,18,762,530,872) : 'test_info',
    (1,18,681,130,726) : 'oil_water_24',
    (1,244,697,415,743) : 'gas_24',
    (1,309,265,347,276) : 'depth_from', 
    (1,531,265,567,276) : 'depth_to',   
    (0,248,208,563,404) : 'md', 
    (2,389,833,413,846) : 'pressure'       
}

def textbox_lookup(file,lookup):
    '''
    Accepts a pdf file and lookup table (see above) and returns a dictionary containing the values of the lookup
    table matched to the text content of textboxes at locations closest to the keys of the the lookup table.
    This is designed to account for slightly shifting text boxes as the content of the forms change.
    '''
    parser = PDFParser(file)
    doc = PDFDocument()
    parser.set_document(doc)
    doc.set_parser(parser)
    doc.initialize('')
    rsrcmgr = PDFResourceManager()
    laparams = LAParams()
    laparams.char_margin = 1.0
    laparams.word_margin = 1.0
    device = PDFPageAggregator(rsrcmgr, laparams=laparams)
    interpreter = PDFPageInterpreter(rsrcmgr, device)
    
    scrape_dict = {}
    score_table = {}
    page_count = -1
    for page in doc.get_pages():
        page_count += 1
        interpreter.process_page(page)
        layout = device.get_result()
        
        #iterate through every object in the pdf layout
        
        for lt_obj in layout:
            if isinstance(lt_obj, LTTextBox) or isinstance(lt_obj, LTTextLine):
                obj_address = (page_count,
                               int(lt_obj.bbox[0]),
                               int(lt_obj.bbox[1]),
                               int(lt_obj.bbox[2]),
                               int(lt_obj.bbox[3]))
                
                for item in lookup.keys():
                    #score based on the distance from a lookup table textbox to the current textbox
                    #the goal is to minimize the score
                    
                    close_score = ((item[0] - obj_address[0])**2*1000000000 +  #Weighted heavily so page is the same
                                   (item[1] - obj_address[1])**2 + 
                                   (item[2] - obj_address[2])**2 +
                                   (item[3] - obj_address[3])**2 +
                                   (item[4] - obj_address[4])**2)
                    if item in score_table.keys():
                        if close_score < score_table[item]:
                            score_table[item] = close_score
                            scrape_dict[lookup[item]] = lt_obj.get_text()
                    else:
                        score_table[item] = close_score
                        scrape_dict[lookup[item]] = lt_obj.get_text()
                        
    return scrape_dict
{% endhighlight %}

## Data Quality Takeaways

I was not able to find any strong relationship between completion design and well performance despite the rigorous data collection effort. I believe this comes down to two issues related to the IP24 metrics gathered from completion reports:

First, when characterizing the success of a completion, we should think about the uplift in production from some theoretical base production value rather than comparing pairs of IPs and completion designs directly. Wells in the dataset came from different benches and over areas of differing rock quality. While I attempted to control for this by selecting wells in a limited area, there was still considerable variation in IPs that could not be explained by completion design. Understanding the landing zones of these wells along with some basic view of type curve areas would have made it easier to make apples-to-apples comparisons that controlled for rock quality.

IP24 is a poor measurement of well performance and is not the metric completions are optimized for. In reality, completions should be optimized for cost, mid-to-long-term well performance and to minimize communication issues between wells. IP24 was the only metric available on a well-by-well basis in Texas (monthly production is prorated on a lease basis), but is ultimately not the correct metric to evaluate.
