# CSV dialect detection with CleverCSV

**Author**: [Gertjan van den Burg](https://gertjan.dev)

In this note we'll show some examples of using CleverCSV, a package for 
handling messy CSV files. We'll start with a motivating example and then show 
some other files where CleverCSV fails.

How does it work? CleverCSV searches the space of all possible dialects of a 
file, and computes a *data consistency measure* that quantifies how much the 
resulting table "looks like real data". The consistency measure combines 
patterns of row lengths in the parsing result and the data type of the 
resulting cells.  This mimicks how a human would identify the dialect. If 
you're wondering why this problem is hard, it's because every dialect will 
give you *some* table, but not necessarily the correct one. More details can 
be found in the paper.

Handy links:

 - [Paper on arXiv](https://arxiv.org/abs/1811.11242)
 - [CleverCSV on GitHub](https://github.com/alan-turing-institute/CleverCSV)
 - [CleverCSV on PyPI](https://pypi.org/project/clevercsv/)
 - [Reproducible Research Repo](https://github.com/alan-turing-institute/CSV_Wrangling/)




We'll compare CleverCSV to the built-in Python CSV module and to Pandas and 
show how these are not as robust as CleverCSV. Note that Pandas always uses 
the comma as separator, unless it is forced to autodetect the dialect, in 
which case it uses the Python Sniffer on the first line (we don't show that 
here).  These files are of course selected for this tutorial, because it 
wouldn't be very interesting to show files where all methods are correct.

We'll define some functions for easy comparisons.




```python
import csv
import clevercsv
import io
import os
import requests
import pandas as pd

from termcolor import colored
from IPython.display import display

def page(url):
    """ Get the content of a webpage using requests, assuming UTF-8 encoding """
    page = requests.get(url)
    content = page.content.decode('utf-8')
    return content

def head(content, num=10):
    """ Preview a CSV file """
    print('--- File Preview ---')
    for i, line in enumerate(io.StringIO(content, newline=None)):
        print(line, end='')
        if i == num - 1:
            break
    print('\n---')

def sniff_url(content):
    """ Utility to run the python Sniffer on a CSV file at a URL """
    try:
        dialect = csv.Sniffer().sniff(content)
        print("CSV Sniffer detected: delimiter = %r, quotechar = %r" % (dialect.delimiter,
                                                                        dialect.quotechar))
    except csv.Error as err:
        print(colored("No result from the Python CSV Sniffer", "red"))
        print(colored("Error was: %s" % err, "red"))

def detect_url(content, verbose=True):
    """ Utility to run the CleverCSV detector on a CSV file at a URL """
    # We have designed CleverCSV to be a drop-in replacement for the CSV module
    try:
        dialect = clevercsv.Sniffer().sniff(content, verbose=verbose)
        print("CleverCSV detected: delimiter = %r, quotechar = %r" % (dialect.delimiter, 
                                                                      dialect.quotechar))
    except clevercsv.Error:
        print(colored("No result from CleverCSV", "red"))

def pandas_url(content):
    """ Wrapper around pandas.read_csv(). """
    buf = io.StringIO(content)
    print(
        "Pandas uses: delimiter = %r, quotechar = %r"
        % (',', '"')
    )
    try:
        df = pd.read_csv(buf)
        display(df.head())
    except pd.errors.ParserError:
        print(colored("ParserError from pandas.", "red"))


def compare(input_, verbose=False, n_preview=10):
    if os.path.exists(input_):
      enc = clevercsv.utils.get_encoding(input_)
      content = open(input_, 'r', newline='', encoding=enc).read()
    else:
      content = page(input_)
    head(content, num=n_preview)
    print("\n1. Running Python Sniffer")
    sniff_url(content)
    print("\n2. Running Pandas")
    pandas_url(content)
    print("\n3. Running CleverCSV")
    detect_url(content, verbose=verbose)
```


### Table embedded in the last record




```python
compare('./data/Table embedded in the last record.csv', n_preview=5)
```

    --- File Preview ---
    id,comment
    1,hello
    2,goodybe
    3,"Some table like:
    id|key|value
    
    ---
    
    1. Running Python Sniffer
    CSV Sniffer detected: delimiter = ',', quotechar = '"'
    
    2. Running Pandas
    Pandas uses: delimiter = ',', quotechar = '"'



<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>comment</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>hello</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>goodybe</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>Some table like:\nid|key|value\n1|apple|10\n2|...</td>
    </tr>
  </tbody>
</table>
</div>


    
    3. Running CleverCSV
    CleverCSV detected: delimiter = ',', quotechar = '"'



You'll notice that all methods says ``,`` is the delimiter and all are ok.

### Pipe character is more frequent than the semicolon




```python
compare('./data/Pipe character is more frequent than the semicoloncsv.csv', n_preview=5)
```

    --- File Preview ---
    1;Orange VUHF;K6CF AnhmHlls|K6COV Orng|K6MWT LkFrstSntg|K6NBR NwprtBch|K6QEH FllrtnRyth|K6SOA LgnBch|K6SOA SnClmnt|K6SOA TrbcCnyn|K6SYU FllrtnSt.J|KA6EEK IrvnSgnlP|KE6FUZ AnhmDsnyl|N6ME Fllrtn|N6SLD LkFrstSntg|W6HBR OrngPlsnts|W6KRW SnClmnt|W6KRW TstnLmRdg|W6VLD HntngtnBch|WA6FV FntnVlly|WA6YNT Plcnt|WB6HRO CstMsCtyH;145.1400|145.1600|145.2200|145.2400|145.2600|145.4000|145.4200|146.0250|146.2650|146.7900|146.8950|146.8950|146.9400|146.9700|147.0600|147.4350|147.4650|147.6450|147.8550|147.9150;144.5400|144.5600|144.6200|144.6400|144.6600|144.8000|144.8200|146.6250|146.8650|146.1900|146.2950|146.2950|146.3400|146.3700|147.6600|146.4000|146.5050|147.0450|147.2550|147.3150;OFF;OFF;OFF;;;OFF;;;Selected;0.5;0.5;0.1;0.1
    
    ---
    
    1. Running Python Sniffer
    CSV Sniffer detected: delimiter = ';', quotechar = '"'
    
    2. Running Pandas
    Pandas uses: delimiter = ',', quotechar = '"'



<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>1;Orange VUHF;K6CF AnhmHlls|K6COV Orng|K6MWT LkFrstSntg|K6NBR NwprtBch|K6QEH FllrtnRyth|K6SOA LgnBch|K6SOA SnClmnt|K6SOA TrbcCnyn|K6SYU FllrtnSt.J|KA6EEK IrvnSgnlP|KE6FUZ AnhmDsnyl|N6ME Fllrtn|N6SLD LkFrstSntg|W6HBR OrngPlsnts|W6KRW SnClmnt|W6KRW TstnLmRdg|W6VLD HntngtnBch|WA6FV FntnVlly|WA6YNT Plcnt|WB6HRO CstMsCtyH;145.1400|145.1600|145.2200|145.2400|145.2600|145.4000|145.4200|146.0250|146.2650|146.7900|146.8950|146.8950|146.9400|146.9700|147.0600|147.4350|147.4650|147.6450|147.8550|147.9150;144.5400|144.5600|144.6200|144.6400|144.6600|144.8000|144.8200|146.6250|146.8650|146.1900|146.2950|146.2950|146.3400|146.3700|147.6600|146.4000|146.5050|147.0450|147.2550|147.3150;OFF;OFF;OFF;;;OFF;;;Selected;0.5;0.5;0.1;0.1</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
</div>


    
    3. Running CleverCSV
    CleverCSV detected: delimiter = '|', quotechar = ''



Sniffer is correct, Pandas Says ``,`` is the delimiter and CleverCSV gets incorrect ``|`` character.




### Undefined field delimiter

The next file is the most difficult: a single-column CSV with an escaped delimiter. The parsers 
must detect the semicolon ``;`` as field delimiter.




```python
compare('./data/Undefined field delimiter.csv', n_preview=5)
```

    --- File Preview ---
    Header
    "This semicolon (;) need to be escaped"
    This CSV has only one column
    We will see if a comma (,) break the logic
    or a single colon (:) or pipe (|)
    
    ---
    
    1. Running Python Sniffer
    [31mNo result from the Python CSV Sniffer[0m
    [31mError was: Could not determine delimiter[0m
    
    2. Running Pandas
    Pandas uses: delimiter = ',', quotechar = '"'
    [31mParserError from pandas.[0m
    
    3. Running CleverCSV
    CleverCSV detected: delimiter = ' ', quotechar = ''



Python Sniffer cannot detect the delimiter. Pandas executes in error and says ``,`` is the delimiter, CleverCSV 
incorrectly obtains the space `` `` character as delimiter. Further more, CleverCSV was unable to detect the quote char!
