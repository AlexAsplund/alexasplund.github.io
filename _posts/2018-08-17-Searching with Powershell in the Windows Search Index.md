---
layout: post
title: Searching with Powershell in the Windows Search Index
---
Happened to go down the rabbit hole and found out that you can search the data stored by Windows Search in powershell.

This can be used on for example a file server, doing more advanced searches, or creates the possibility for you to easily search for metadata about your pictures.

I created an ugly function for it:

## The function
```powershell
Function Find-WindowsSearchFiles {
    param(
        [parameter(Mandatory)]
        [string]$SearchString,

        [parameter(Mandatory)]
        [string]$Path,

        $SearchProperty = "System.FileName",
        $SearchOperator = "=",
        [Array]$Properties = @("System.ItemName", "System.DateCreated", "System.DateModified", "System.Size", "System.ItemType")

    )

    $Columns = $Properties -Join ","

    $Query = "
        SELECT $Columns  FROM SYSTEMINDEX
        WHERE $SearchString AND SCOPE = '$Path'
    "

    $ConnectionString = "provider=search.collatordso;extended properties=’application=windows’;" 
    $OleDBAdapter = [System.Data.OleDb.OleDbDataAdapter]::New($Query, $ConnectionString)
    $Dataset = [System.Data.DataSet]::New()

    if ($OleDBAdapter.fill($Dataset)) {
        $Dataset.tables[0]
    }

}
```

## How to use it

First, find out what information you want about the files and put it in an array.

Column names are found here: https://gist.github.com/AlexAsplund/67a1944dd1309f68cc73cc9ed566bfc9

```powershell
$Properties = @(

    'System.Author',
    'System.Category',
    'System.Comment',
    'System.Company',
    'System.ComputerName',
    'System.ContentStatus',
    'System.ContentType',
    'System.Copyright',
    'System.DateAccessed',
    'System.DateAcquired',
    'System.DateArchived',
    'System.DateCompleted',
    'System.DateCreated',
    'System.DateImported',
    'System.DateModified',
    'System.DueDate',
    'System.EndDate',
    'System.FileAttributes',
    'System.FileDescription',
    'System.FileExtension',
    'System.FileName',
    'System.FileOwner',
    'System.ItemAuthors',
    'System.ItemDate',
    'System.ItemFolderNameDisplay',
    'System.ItemFolderPathDisplay',
    'System.ItemFolderPathDisplayNarrow',
    'System.ItemName',
    'System.ItemNameDisplay',
    'System.ItemNamePrefix',
    'System.ItemParticipants',
    'System.ItemPathDisplay',
    'System.ItemPathDisplayNarrow',
    'System.ItemType',
    'System.ItemTypeText',
    'System.ItemUrl',
    'System.Keywords',
    'System.Kind',
    'System.SourceItem',
    'System.StartDate',
    'System.Status',
    'System.Subject',
    'System.ThumbnailCacheId',
    'System.Title',
    'System.Document.ByteCount',
    'System.Document.CharacterCount',
    'System.Document.ClientID',
    'System.Document.Contributor',
    'System.Document.DateCreated',
    'System.Document.DatePrinted',
    'System.Document.DateSaved',
    'System.Document.Division',
    'System.Document.DocumentID',
    'System.Document.HiddenSlideCount',
    'System.Document.LastAuthor',
    'System.Document.LineCount',
    'System.Document.Manager',
    'System.Document.PageCount',
    'System.Document.ParagraphCount',
    'System.Document.PresentationFormat',
    'System.Document.RevisionNumber',
    'System.Document.SlideCount',
    'System.Document.TotalEditingTime',
    'System.Document.WordCount',
    'System.Document.ByteCount',
    'System.Document.CharacterCount',
    'System.Document.ClientID',
    'System.Document.Contributor',
    'System.Document.DateCreated',
    'System.Document.DatePrinted',
    'System.Document.DateSaved',
    'System.Document.Division',
    'System.Document.DocumentID',
    'System.Document.HiddenSlideCount',
    'System.Document.LastAuthor',
    'System.Document.LineCount',
    'System.Document.Manager',
    'System.Document.PageCount',
    'System.Document.ParagraphCount',
    'System.Document.PresentationFormat',
    'System.Document.RevisionNumber',
    'System.Document.SlideCount',
    'System.Document.TotalEditingTime',
    'System.Document.WordCount',
    'System.Search.AutoSummary',
    'System.Search.Contents',
    'System.Search.EntryID',
    'System.Search.GatherTime',
    'System.Search.Rank',
    'System.Search.Store'

)
```

Create a searchstring and point the function to a directory:

```powershell
> Find-WindowsSearchFiles -SearchString "System.Author LIKE '%John Doe%'" -Path 'F:\Files\' -Properties $Properties 

Output:
SYSTEM.AUTHOR                       : {John Doe}
SYSTEM.CATEGORY                     : 
SYSTEM.COMMENT                      : 
SYSTEM.COMPANY                      : 
SYSTEM.COMPUTERNAME                 : WS103947
SYSTEM.CONTENTSTATUS                : 
SYSTEM.CONTENTTYPE                  : application/vnd.openxmlformats-officedocument.wordprocessingml.document
SYSTEM.COPYRIGHT                    : 
SYSTEM.DATEACCESSED                 : 2018-06-20 16:54:59
SYSTEM.DATEACQUIRED                 : 
SYSTEM.DATEARCHIVED                 : 
SYSTEM.DATECOMPLETED                : 
SYSTEM.DATECREATED                  : 2018-05-12 19:04:47
SYSTEM.DATEIMPORTED                 : 2018-05-12 19:04:48
SYSTEM.DATEMODIFIED                 : 2018-06-20 16:54:59
SYSTEM.DUEDATE                      : 
SYSTEM.ENDDATE                      : 
SYSTEM.FILEATTRIBUTES               : 32
SYSTEM.FILEDESCRIPTION              : 
SYSTEM.FILEEXTENSION                : .docx
SYSTEM.FILENAME                     : PrinterJanitor.docx
SYSTEM.FILEOWNER                    : domain\username
SYSTEM.ITEMAUTHORS                  : {John Doe}
SYSTEM.ITEMDATE                     : 2018-06-20 16:54:00
SYSTEM.ITEMFOLDERNAMEDISPLAY        : docs
SYSTEM.ITEMFOLDERPATHDISPLAY        : F:\Files\Documents
SYSTEM.ITEMFOLDERPATHDISPLAYNARROW  : docs (F:\Files\Documents)
SYSTEM.ITEMNAME                     : DocumentName.docx
SYSTEM.ITEMNAMEDISPLAY              : DocumentName.docx
SYSTEM.ITEMNAMEPREFIX               : 
SYSTEM.ITEMPARTICIPANTS             : {John Doe}
SYSTEM.ITEMPATHDISPLAY              : F:\Files\Documents\DocumentName.docx
SYSTEM.ITEMPATHDISPLAYNARROW        : PrinterJanitor (F:\Files\Documents)
SYSTEM.ITEMTYPE                     : .docx
SYSTEM.ITEMTYPETEXT                 : Microsoft Word Document
SYSTEM.ITEMURL                      : file:F:\Files\Documents\DocumentName.docx
SYSTEM.KEYWORDS                     : 
SYSTEM.KIND                         : {document}
SYSTEM.SOURCEITEM                   : 
SYSTEM.STARTDATE                    : 
SYSTEM.STATUS                       : 
SYSTEM.SUBJECT                      : 
SYSTEM.THUMBNAILCACHEID             : 9756032152378704838
SYSTEM.TITLE                        : My Very Important Document
SYSTEM.DOCUMENT.BYTECOUNT           : 
SYSTEM.DOCUMENT.CHARACTERCOUNT      : 5994
SYSTEM.DOCUMENT.CLIENTID            : 
SYSTEM.DOCUMENT.CONTRIBUTOR         : 
SYSTEM.DOCUMENT.DATECREATED         : 2018-05-12 18:55:00
SYSTEM.DOCUMENT.DATEPRINTED         : 
SYSTEM.DOCUMENT.DATESAVED           : 2018-06-20 16:54:00
SYSTEM.DOCUMENT.DIVISION            : 
SYSTEM.DOCUMENT.DOCUMENTID          : 
SYSTEM.DOCUMENT.HIDDENSLIDECOUNT    : 
SYSTEM.DOCUMENT.LASTAUTHOR          : John Doe
SYSTEM.DOCUMENT.LINECOUNT           : 49
SYSTEM.DOCUMENT.MANAGER             : 
SYSTEM.DOCUMENT.PAGECOUNT           : 6
SYSTEM.DOCUMENT.PARAGRAPHCOUNT      : 14
SYSTEM.DOCUMENT.PRESENTATIONFORMAT  : 
SYSTEM.DOCUMENT.REVISIONNUMBER      : 7
SYSTEM.DOCUMENT.SLIDECOUNT          : 
SYSTEM.DOCUMENT.TOTALEDITINGTIME    : 16200000000
SYSTEM.DOCUMENT.WORDCOUNT           : 1051
SYSTEM.DOCUMENT.BYTECOUNT1          : 
SYSTEM.DOCUMENT.CHARACTERCOUNT1     : 5994
SYSTEM.DOCUMENT.CLIENTID1           : 
SYSTEM.DOCUMENT.CONTRIBUTOR1        : 
SYSTEM.DOCUMENT.DATECREATED1        : 2018-05-12 18:55:00
SYSTEM.DOCUMENT.DATEPRINTED1        : 
SYSTEM.DOCUMENT.DATESAVED1          : 2018-06-20 16:54:00
SYSTEM.DOCUMENT.DIVISION1           : 
SYSTEM.DOCUMENT.DOCUMENTID1         : 
SYSTEM.DOCUMENT.HIDDENSLIDECOUNT1   : 
SYSTEM.DOCUMENT.LASTAUTHOR1         : John Doe
SYSTEM.DOCUMENT.LINECOUNT1          : 49
SYSTEM.DOCUMENT.MANAGER1            : 
SYSTEM.DOCUMENT.PAGECOUNT1          : 6
SYSTEM.DOCUMENT.PARAGRAPHCOUNT1     : 14
SYSTEM.DOCUMENT.PRESENTATIONFORMAT1 : 
SYSTEM.DOCUMENT.REVISIONNUMBER1     : 7
SYSTEM.DOCUMENT.SLIDECOUNT1         : 
SYSTEM.DOCUMENT.TOTALEDITINGTIME1   : 16200000000
SYSTEM.DOCUMENT.WORDCOUNT1          : 1051
SYSTEM.SEARCH.AUTOSUMMARY           : 
SYSTEM.SEARCH.CONTENTS              : 
SYSTEM.SEARCH.ENTRYID               : 49408
SYSTEM.SEARCH.GATHERTIME            : 2018-06-20 16:55:00
SYSTEM.SEARCH.RANK                  : 1000
SYSTEM.SEARCH.STORE                 : file
```



****
----