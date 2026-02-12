--- 
layout: post 
title: LiveOps Localization System for Your Game 
date: 2022-01-15 20:32:20 +0300 
tags: [csharp, tools, python3, aws cloud, unity] 
--- 

### **Purpose** 
This text aims to talk about the implementation of location systems in digital games. There is no intention to talk about how to write localized text. 

### **Problem** 
Imagine adding thousands of lines and texts from a game with several languages (Russian, Spanish, Portuguese, etc.). A work that this results in is gigantic. The game Frostpunk has hundreds of phrases and several languages, the effort required is very big. Another game in the same situation is Horizon Chase Turbo (I'm currently working on it with the Aquiris team), but it used a solution that makes this task less arduous and less chance of error! 

How does the game dynamically change languages? How does the game change all the text with just one user input? What if developers want to change some phrase in a specific language in a file with thousands and thousands of characters? How to find this particular phrase? All these problems are smoothed out using the following technique. 

### **How to Solve That?** 
(We performed an implementation in the Unity game development engine) 

Several ways to localize our game exist, using .xml, .json files, your file pattern, among many others. Editing these types of files is not intuitive, is repetitive, and with a large margin for error. However, in my opinion, the best way to accomplish this task is what I'm going to teach you here, using Google's great tool, Google Sheets, and .csv files! 

### **LiveOps** 
I'll show you how it's possible to do any text update in a LiveOps way without having to recompile your entire game, create a new build and submit it to some platform, giving us the freedom to update some text in less than 10 minutes, 100% LiveOps. 

The file path follows this flow: Google Sheets → AWS Lambda → AWS S3 → StackPath CDN → Unity Game Client.

### **Technologies** 
- Unity 
- AWS Lambdas 
- AWS S3 
- StackPath 
- C# 
- Python3 

# Why Use this Technique? 

### **Anyone can edit the localization** 
Not only programmers can edit localization files. Anyone can it's simple to edit strings within the game. It is no longer a complex and dangerous file. It becomes a spreadsheet that is easy to access and modify. 

### **It is possible to automate repetitive texts** 
With a little more advanced knowledge of spreadsheets and Google Sheet formulas, you can automate various texts. No more writing "speech_99", "speech_100" for your text key, do it with a Google Sheets formula. 

### **Finding and editing an error is much easier** 
We will see at the time of creating our spreadsheet how to make the margin of error miniscule. 

### **Promotional texts anytime** 
By adding the LiveOps system, it's possible to build a "What's New" for your game, for example. In which you can put any text. Encouraging the purchase of some DLC or something! All this without any build, recompilation, etc. 

# Creating the Spreadsheet 

### **Struct** 
The structure is simple. The "header" of the spreadsheet has the access key and its corresponding languages. For example, the first column is labeled "KEY", followed by columns for each language: "en", "es", "pt", etc. Each row then contains the key identifier and its translations across all languages.

### **Formulas** 
To avoid some errors, some formulas were created to identify them: 

a. **Empty Cell** 
An empty cell can harm when our Parser reads the .csv, so, to avoid this type of error, we add a Conditional Format to check if there is any and if so, we paint it red. Apply this formula to all language columns: `=ISBLANK(B2)` where B2 represents each cell in your language columns.

b. **Cell with "enter" at the end** 
Another thing that can hurt when Parser reads the .csv is the "enter" at the end of the cell. Instead, we use "\n" to prevent another Conditional Format is added. Use this formula: `=RIGHT(B2,1)=CHAR(10)` to detect line breaks at the end of cells.

c. **Cell with space at the end or beginning** 
Unnecessary characters are common. Space at the end and beginning of the cell, to prevent another Conditional Format is added. Apply this formula: `=OR(LEFT(B2,1)=" ", RIGHT(B2,1)=" ")` to highlight cells with leading or trailing spaces.

### **Permissions** 
Set your Google Sheet permissions to "Anyone on the internet with this link can view". This can be configured in the Share settings of your Google Sheet by selecting "Change to anyone with the link" and setting the permission level to "Viewer".

# Creating Bucket on AWS 

This AWS bucket stores the Localization .csv. This .csv comes from a Lambda that we will create in the future, which will feed on our Google Sheets spreadsheet and save the .csv here! 

1. Enable bucket ACL 
2. Does not block all public access 

# Creating Lambda 

After carrying out the previous steps, we need a way to have a version of our spreadsheet in AWS S3 in the correct file format, for we are going to use lambdas AWS 

### **Why AWS Lambdas?** 
a. There's No Infrastructure to Manage 
b. AWS Lambda Has Strong Security Support 
c. AWS Lambdas Are Event Driven 
d. You Only Pay for What You Use 

To create our lambda we use the lambdas system offered by AWS, and python3 (Teaching how to create it in [previous post](https://gabrielprzybysz.github.io/gabeblog/forant/) on "Lambdas" section). The lambda created below can be called by some API or scheduler to constantly update localization. 

First of all, the packages used are: 

```python
import json
import urllib3
import boto3
```

After importing the packages, let's declare some constant variables: 

```python
CSV_DOWNLOAD_LINK = "https://docs.google.com/spreadsheets/d/{YOUR SHEET ID}/gviz/tq?tqx=out:csv"
```

To find your sheet ID, look at your Google Sheets URL. The sheet ID is the long string of characters between "/d/" and "/edit" in the URL. For example, in the URL `https://docs.google.com/spreadsheets/d/1ABC...XYZ/edit`, the sheet ID would be `1ABC...XYZ`.

Now let's declare the name of the Bucket created on AWS: 

```python
BUCKET_NAME = 'yourgame-localization'
```

And finally, the name of the file created inside the bucket (the .csv): 

```python
FILE_NAME = 'localization.csv'
```

Now, first of all, let's download the .csv hosted on Google Drive (The spreadsheet itself): 

```python
def download_csv():
    http = urllib3.PoolManager()
    resp = http.request("GET", CSV_DOWNLOAD_LINK)
    return resp.data
```

And then with this binary data downloaded from Google Drive we will create a new .csv file in AWS: 

```python
def create_file_in_s3():
    binary_csv = download_csv()
    s3 = boto3.resource("s3")
    s3.Bucket(BUCKET_NAME).put_object(ACL='public-read-write', Key=FILE_NAME, Body=binary_csv)
```

# .csv Unity Parser 

Now in Unity, we need to find a way to use the created file. It needs to be easily accessible by the UI system. An example: LocalizationController.Localize(string id) - the function that returns a localized string for the current language selected by the player, thus making all texts dynamically change. 

To start, we need an async download of our localization file hosted on AWS to occur. For this reason, we created the LocalizationDownloader.cs script. 

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Networking;

public class LocalizationDownloader : MonoBehaviour
{
    private const string CSV_URL = "";
    
    private static string _rawLocalizationCsv;
    private static readonly int Loading = Animator.StringToHash("loading");
    
    [SerializeField] private Animator _loadingAnimator;
    
    public static string RawLocalization => _rawLocalizationCsv;
    public static bool IsLoading = true;
    
    void Awake()
    {
        DontDestroyOnLoad(this);
        StartCoroutine(DownloadLocalization());
    }
    
    private IEnumerator DownloadLocalization()
    {
        using (UnityWebRequest client = UnityWebRequest.Get(CSV_URL))
        {
            UnityWebRequestAsyncOperation result = client.SendWebRequest();
            
            SetLoad(true);
            yield return new WaitUntil(() => result.isDone);
            SetLoad(false);
            
            _rawLocalizationCsv = result.webRequest.downloadHandler.text;
        }
        
        LocalizationController.Instance.Initialize();
    }
    
    private void SetLoad(bool isLoading)
    {
        _loadingAnimator.SetBool(Loading, isLoading);
        IsLoading = isLoading;
    }
}
```

So the game does not start without localization. A loading system was implemented, as seen in the previous script. 

Now with the .csv string downloaded, we need to handle this data. For this, we will use a library called CSVHelper. It will allow us to work with csv in a more optimized way. 

The CSVHelper library can be downloaded following the following Microsoft tutorial (dlls can be found inside my project at [git](https://github.com/GabrielPrzybysz/liveops-localization-system)): 
https://docs.microsoft.com/en-us/visualstudio/gamedev/unity/unity-scripting-upgrade 

With the library installed, let's create a controller to merge this library with the previously downloaded data. For example purposes, the controller implementation used the singleton pattern. I believe it could be done in a "cleaner" way. The script can be found in the repository. 

How to merge downloaded data with the CSVHelper library? 

```csharp
private void LoadLocalizationFromCSV()
{
    CsvConfiguration csvConfiguration = new CsvConfiguration(CultureInfo.CurrentCulture)
    {
        HasHeaderRecord = false,
        Delimiter = ","
    };
    
    using (var csvParser = new CsvParser(new StringReader(LocalizationDownloader.RawLocalization), csvConfiguration))
    {
        using (var csvReader = new CsvReader(csvParser))
        {
            try
            {
                var localizationSheet = csvReader.GetRecords<RawLocalizationItem>().ToList();
                
                foreach (var rawItem in localizationSheet)
                {
                    _localizedItems.Add(rawItem.KEY, new LocalizationItem(rawItem.en, rawItem.es, rawItem.pt));
                }
            }
            catch (Exception e)
            {
                Console.WriteLine(e.Message);
                throw;
            }
        }
    }
}
```

It needs to be easily accessible by the UI system and simple to use, so now: 

```csharp
public string Localize(string key)
{
    switch (CurrentLanguage)
    {
        case Languages.EN:
            return _localizedItems[key].En;
        case Languages.ES:
            return _localizedItems[key].Es;
        case Languages.PT:
            return _localizedItems[key].Pt;
        default:
            throw new ArgumentOutOfRangeException();
    }
}
```

Now, with all that ready, let's create the script that will be added to the text element to be located: 

```csharp
public class TextLocalize : MonoBehaviour
{
    [SerializeField] private string _keyToLocalize;
    
    private void OnEnable()
    {
        gameObject.GetComponent<Text>().text = LocalizationController.Instance.Localize(_keyToLocalize);
    }
}
```

Example of use: Add the TextLocalize component to any Unity Text GameObject, then set the "_keyToLocalize" field in the Inspector to the key you want to display (such as "greeting_hello" or "menu_start"). When the GameObject is enabled, the text will automatically update to show the localized version based on the current language setting.

# Thanks 

With this, I hope to make it easier for game developers to create a localization system for their games. I'm a Computer Science student with experience in the gaming industry, and here I showed some techniques I learned. If you have any questions or improvements, send me an email, or access any of my other social networks (they can be found here in the lower-left corner). 

**Author**: Gabriel Przybysz Gonçalves Júnior - Backend Programmer
