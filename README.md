# webcrawl-basic.sas

/* Could use this, might be slower/less robust */
* filename src url "https://wwwn.cdc.gov/nndss/conditions/search/";
 
/* I prefer PROC HTTP for speed and flexibility */
filename src temp;
proc http
 method="GET"
 url="https://wwwn.cdc.gov/nndss/conditions/search/"
 out=src;
run;

<tr >
	<td style="text-align:left;vertical-align:middle;">
    	<a href="/nndss/conditions/chancroid/">
		Chancroid
		</a>
	</td>
	<td class="tablet-hidden"></td>
	<td class="tablet-hidden"></td>
	<td class="tablet-hidden"><i>Haemophilus ducreyi</i></td>
	<td  >1944 </td>
	<td  > Current</td>
</tr>

/* Read the entire file and skip the blank lines */
/* the LEN indicator tells us the length of each line */
data rep;
infile src length=len lrecl=32767;
input line $varying32767. len;
 line = strip(line);
 if len>0;
run;
 
/* Parse the lines and keep just condition names */
/* When a condition code is found, grab the line following (full name of condition) */
/* and the 8th line following (Notification To date)                                */
/* Relies on this page's exact layout and line break scheme */
data parsed (keep=condition_code condition_full note_to);
 length condition_code $ 40 condition_full $ 60 note_to $ 20;
 set rep;
 if find(line,"/nndss/conditions/") then do;
   condition_code=scan(line,4,'/');
   pickup= _n_+1 ;
   pickup2 = _n_+8;
   /* "read ahead" one line */
   set rep (rename=(line=condition_full)) point=pickup;
   condition_full = strip(condition_full);
 
   /* "read ahead" 8 lines */
   set rep (rename=(line=note_to)) point=pickup2;
   /* use SCAN to get just the value we want */
   note_to = strip(scan(note_to,2,'<>'));
 
   /* write this record */
   output;
  end;
run;

/* Create a temp folder to hold the HTML files */
options dlcreatedir;
%let htmldir = %sysfunc(getoption(WORK))/html;
libname html "&htmldir.";
libname html clear;
 
/* collect the first few pages of results of this site */
%macro getPages(num=);
 %do i = 1 %to &num.;
   %let url = https://communities.sas.com/t5/custom/page/page-id/activity-hub?page=&i.;
   filename dest "&htmldir./page&i..html";
   proc http 
      method="GET"
	  url= "&url."
	  out=dest;
   run;
 %end;
%mend;
 
/* How many pages to collect */
%getPages(num=3);
 
/* Use the wildcard feature of INFILE to read all */
/* matching HTML files in a single step */
data results;
 infile "&htmldir./*.html" length=len lrecl=32767;
 input line $varying32767. len ;
 line = strip(line);
 if len>0;
run;



# webcrawl-basic.sas - another one
data work.links_to_crawl;
 length url $256 ;
 input url $;
 datalines;
http://www.yahoo.com
http://www.sas.com
http://www.google.com
;
run;

 data work.links_crawled;
 length url $256;
 run;

/* pop the next url off */
 %let next_url = ;
 data work.links_to_crawl;
 set work.links_to_crawl;
 if _n_ eq 1 then call symput("next_url", url);
 else output;
 run;

/* crawl the url */
 filename _nexturl url "&next_url";

/* put the file we crawled here */
 filename htmlfile "url_file.html";
 
 /* find more urls */
 data work._urls(keep=url);
 length url $256 ;
 file htmlfile;
 infile _nexturl length=len;
 input text $varying2000. len;
 put text;
 start = 1;
 stop = length(text);
 
 if _n_ = 1 then do;
 retain patternID;
 pattern = '/href="([^"]+)"/i';
 patternID = prxparse(pattern);
 end;
 
 call prxnext(patternID, start, stop, text, position, length);

 do while (position ^= 0);
 url = substr(text, position+6, length-7);
 output;
 call prxnext(patternID, start, stop, text, position, length);
 end;
run;

 /* add the current link to the list of urls we have already crawled */
 data work._old_link;
 url = "&next_url";
 run;
 proc append base=work.links_crawled data=work._old_link force;
 run;
 
 /*
* only add urls that we have not already crawled
* or that are not queued up to be crawled
*
*/
 proc sql noprint;
 create table work._append as
 select url
 from work._urls
 where url not in (select url from work.links_crawled)
 and url not in (select url from work.links_to_crawl);
 quit;

 /* add new links */
 proc append base=work.links_to_crawl data=work._append force;
 run;


