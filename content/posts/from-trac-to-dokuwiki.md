Title: From Trac to Dokuwiki
Date: 2008-08-13 23:27:51
Category: English
Tags: trac, dokuwiki
Author: frommelmak

This is my first post in English, so I apologize in advance for my poor grammar.

I'll try to explain how to export all the pages of a Trac wiki into a Dokuwiki . All the process is based in my own shell script, which uses trac-admin, wget and html2wiki to export all the content of your Trac wiki into a dokuwiki folders.

At a glance, the main steps performed by the script are the following:

  * Get the Trac wiki pages as a HTML files using `wget`.
  * Remove all the HTML code belonging to the Trac page itself.
  * Convert all the pages from HTML to Dokuwiki format (using `html2wiki`).
  * Fix some internal/media links and transform footer attachments into dokuwiki format.
  * Rename all the pages and media from upper to lower case.

The output of the script are two tarball files. One with the Dokuwiki pages, and the oter one with the media files.

As you probably know, the pages in a Dokuwiki are stored in simple text files,using the Dokuwiki format. All these pages are stored inside a folder called pages, using different subfolders for every namespace on the wiki. The same occour with the media files. All of them are stored inside folders named as the namespace they belong to, beneath the media folder. So, you just need to extract the content of the media tarball into the media folder and the content of the pages tarball inside the pages folder. And that's all, your pages are now stored inside a Dokuwiki.

If you want to see the script, click on the Leer ms link below.


    
        
    :::bash
    # Marcos Martinez
    #
    # Dependencies: html2wiki
    # trac-admin
    # wget
    #
    # Installation of html2wiki: perl -MCPAN -e 'install HTML::WikiConverter::DokuWiki'
    #
      
    TRAC_PROJECT_ENV='/path/to/trac/project'
    TRAC_HOSTNAME='x.x.x.x or URL'
    TRAC_PORT='80'
        
    # Fill up the following two lines if your trac is using basic athentication method.
    USER=your_trac_user
    PASS=your_trac_password
           
    # By default, all the pages will be exported inside a dokuwiki namespace named as the Trac Project
    DOKUWIKI_NAMESPACE=$(grep '^name' $TRAC_PROJECT_ENV/conf/trac.ini | gawk '{print $3}')
    #DOKUWIKI_NAMESPACE='your_desired_dokuwiki_namespace'
           
    TRAC_BASE_NAME=$(grep 'base_url =' $TRAC_PROJECT_ENV/conf/trac.ini | gawk '{print $3}')
    TRAC_URL_LOGIN=http://$TRAC_HOSTNAME:$TRAC_PORT$TRAC_BASE_NAME/login
    TRAC_URL_WIKI=http://$TRAC_HOSTNAME:$TRAC_PORT$TRAC_BASE_NAME/wiki
    TRAC_PROJECT_NAME=$(grep '^name' $TRAC_PROJECT_ENV/conf/trac.ini | gawk '{print $3}')
    TRAC_INSTALL_DATE=$(ls -l $TRAC_PROJECT_ENV/README |gawk '{print $6   $7}')
    TRAC_WIKI_PAGES=$(echo $TRAC_BASE_NAME/wiki | sed 's/\//\\\//g')
    TRAC_WIKI_ATTACHMENTS=$(echo $TRAC_BASE_NAME/attachment/wiki| sed 's/\//\\\//g')
    TRAC_WIKI_RAW_ATTACHMENTS=$(echo $TRAC_BASE_NAME/raw-attachment/wiki | sed 's/\//\\\//g')
    TRAC_BS=$(echo $TRAC_BASE_NAME | sed 's/\//\\\//g')
            
    # The PLIST var has the list of pages we want to export.
    # By default, all the wiki pages modified after the installation date
    # will be exported.
    PLIST=$(trac-admin $TRAC_PROJECT_ENV wiki list|grep -v $TRAC_INSTALL_DATE | gawk '{print $1}'|sed '1,3d')
            
    # Starting authenticated session
    wget --quiet --spider \
         --user=$USER --password=$PASS \
         --save-cookies galletitas \
         --keep-session-cookies $TRAC_URL_LOGIN
            
    mkdir $DOKUWIKI_NAMESPACE
            
    for PAGE in $PLIST
    do
    echo Downloading: $PAGE
      wget --load-cookies galletitas --quiet -P $DOKUWIKI_NAMESPACE $TRAC_URL_WIKI/$PAGE
    done
          
          
    for FILE in `ls $DOKUWIKI_NAMESPACE`
    do
      echo Parsing $FILE
            
      # Removing standard HTML trac code (header and footer)
      sed -i '1,66d' $DOKUWIKI_NAMESPACE/$FILE
      sed -i -n -e :a -e '1,45!{P;N;D;};N;ba' $DOKUWIKI_NAMESPACE/$FILE
            
      # Removing table of contents (TOC) because in dokuwiki, the TOC is dynamically created at runtime.
      sed -i '/class=\wiki-toc\/d' $DOKUWIKI_NAMESPACE/$FILE
            
      # Adding HTML headers
      sed -i '1i\html\nhead\n/head\nbody' $DOKUWIKI_NAMESPACE/$FILE
      sed -i '$a\/body\n/html' $DOKUWIKI_NAMESPACE/$FILE
      html2wiki --dialect DokuWiki $DOKUWIKI_NAMESPACE/$FILE  $DOKUWIKI_NAMESPACE/$FILE.txt
      rm $DOKUWIKI_NAMESPACE/$FILE
           
      # Fixing internal links
      sed -i s/\[\[$TRAC_WIKI_PAGES\//\[\[$DOKUWIKI_NAMESPACE:/g $DOKUWIKI_NAMESPACE/$FILE.txt
            
      # Fixing media links
      sed -i s/\[\[$TRAC_WIKI_ATTACHMENTS\/$FILE\//{{:$DOKUWIKI_NAMESPACE:/g $DOKUWIKI_NAMESPACE/$FILE.txt
      sed -i s/{{$TRAC_WIKI_RAW_ATTACHMENTS\/$FILE\///g $DOKUWIKI_NAMESPACE/$FILE.txt
      sed -i s/}}]]/}}/gDOKUWIKI_NAMESPACE/$FILE.txt
            
      # Removing timeline data from 'Attachments' at the EOF if needed
            
      NUM=$(grep -n '==== Attachments ====' $DOKUWIKI_NAMESPACE/$FILE.txt | cut -d':' -f1)
      grep -q '==== Attachments ====' $DOKUWIKI_NAMESPACE/$FILE.txt  sed -i $NUM,\$s/\[\[$TRAC_BS\/timeline?from=.*]] ago.//g $DOKUWIKI_NAME            SPACE/$FILE.txt
      # Formating attachments footer as Dokuwiki linkonly links
      grep -q '==== Attachments ====' $DOKUWIKI_NAMESPACE/$FILE.txt  sed -i $NUM,\$s/ * {{/{{/ $DOKUWIKI_NAMESPACE/$FILE.txt
      grep -q '==== Attachments ====' $DOKUWIKI_NAMESPACE/$FILE.txt  sed -i $NUM,\$s/|/?linkonly|/ $DOKUWIKI_NAMESPACE/$FILE.txt
      grep -q '==== Attachments ====' $DOKUWIKI_NAMESPACE/$FILE.txt  sed -i $NUM,\$s/]]/}}/ $DOKUWIKI_NAMESPACE/$FILE.txt
            
      # In Dokuwiki, only lower characters are allowed in filenames
      mv -u $DOKUWIKI_NAMESPACE/$FILE.txt `echo $DOKUWIKI_NAMESPACE/$FILE.txt| tr [:upper:] [:lower:]` 2/dev/null
            
    done
            
    tar -czvf $DOKUWIKI_NAMESPACE.tar.gz $DOKUWIKI_NAMESPACE
    mkdir $DOKUWIKI_NAMESPACE-media
    find $TRAC_PROJECT_ENV/attachments/wiki -type f -exec cp {} $DOKUWIKI_NAMESPACE-media \;
            
    for FILE in `ls $DOKUWIKI_NAMESPACE-media`
    do
      # In Dokuwiki, only lower characters are allowed in media filenames
      mv -u $DOKUWIKI_NAMESPACE-media/$FILE `echo $DOKUWIKI_NAMESPACE-media/$FILE| tr [:upper:] [:lower:]` 2/dev/null
    done
            
    tar -czvf $DOKUWIKI_NAMESPACE-media.tar.gz $DOKUWIKI_NAMESPACE-media
            
    rm -rf $DOKUWIKI_NAMESPACE-media $DOKUWIKI_NAMESPACE
    rm galletitas             
            
