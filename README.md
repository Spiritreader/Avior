THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.


- [Requirements](#requirements)
- [Setup](#setup)
  * [Basic Installation](#basic-installation)
  * [DVB Media Server Configuration](#dvb-media-server-configuration)
  * [Windows](#windows)
- [Introduction](#introduction)
    + [Avior is a two component software](#avior-is-a-two-component-software)
    + [What does Avior do?](#what-does-avior-do-)
    + [What does Infuser do?](#what-does-infuser-do-)
    + [Workflow Example](#workflow-example)
- [Files, Formats, Configuration](#files--formats--configuration)
  * [toCode.txt](#tocodetxt)
  * [main.cfg:](#maincfg-)
      - [Tags](#tags)
      - [General configuration](#general-configuration)
  * [encoderConfig folder:](#encoderconfig-folder-)
      - [Handbrake Variables for tag.cfg:](#handbrake-variables-for-tagcfg-)
      - [ffmpeg variables for tag.cfg:](#ffmpeg-variables-for-tagcfg-)
      - [File output for tag.cfg](#file-output-for-tagcfg)
  * [Encoder selection](#encoder-selection)
  * [Encoder selection mechanism](#encoder-selection-mechanism)
  * [directories.cfg](#directoriescfg)
  * [File name construction & File handling](#file-name-construction---file-handling)
      - [subtitlesToCut.txt and namesToCut.txt](#subtitlestocuttxt-and-namestocuttxt)
  * [Default behavior for existing files:](#default-behavior-for-existing-files-)
  * [Default behavior for encoded files](#default-behavior-for-encoded-files)
  * [Intelligent Re-Encode Mechanism:](#intelligent-re-encode-mechanism-)
    + [Logfile fulltext search](#logfile-fulltext-search)
        * [How it works](#how-it-works)
        * [Supported logfile types](#supported-logfile-types)
    + [Intelligent Audio Comparator](#intelligent-audio-comparator)
      - [Algorithm](#algorithm)
      - [audioFormat Folder](#audioformat-folder)
    + [Size estimation](#size-estimation)
        * [Calculation of maximum Length](#calculation-of-maximum-length)
        * [Calculation and positioning of re-encode slices](#calculation-and-positioning-of-re-encode-slices)
      - [Example of how IREM looks like in the log file for an error case:](#example-of-how-irem-looks-like-in-the-log-file-for-an-error-case-)
  * [Logfiles](#logfiles)
- [Database with Infuser and Avior](#database-with-infuser-and-avior)
  * [Avior database client management](#avior-database-client-management)
  * [How Infuser selects an eligible client for a job](#how-infuser-selects-an-eligible-client-for-a-job)


Requirements
======

* Java 8
* Windows 7 and higher
* Handbrake + HandbrakeCLI (HB Version 0.9 or lower)
* HandbrakeCLI (HB Version 1.0 and higher)
* DVB Media Server (any version) with logfiles ticked
* Most recent dotNET framework for your operating system
* ffmpeg.exe (most recent version) dropped into the same directory as VDRAvior.jar
* Optional: MongoDB for storing jobs in a database

Setup
======

## Basic Installation

Extract all contents into a directory of your choice.

## DVB Media Server Configuration

Parameter Tasks ("Aufgaben") for Recording Service:
{SOURCE_FILE}" "{TITLE}" "{SUBTITLE}"
Select an executable and choose "Recording Infuser Windows.exe"

Also make sure logfiles are ticked in the main configuration.

More detailed setup in the future

## Windows

Add new Windows Environment User Variable: 'HandbrakeCLI' with the Path to the Handbrake folder

Copy your version of ffmpeg.exe into the same directory where VDRAvior.jar resides in. I won't include ffmpeg because it's a licensing minefield.

Introduction
======

Here's what you can do with the Software.

The VDRAvior, or Video Data Recorder Auto Encode is a tool to automatically encode video files recorded by DVBRecording Service,
renaming them to a mostly IMDB compatible name (which can later be movie scraped with a media center, i.e Kodi).

#### Avior is a two component software
* VDRAvior.jar (Main application)
* Recording Infuser Windows.exe (gets called from DVBRecSvc)

__The charset / codepage is essential for the functionality of the program!__

Its folder structure consists of the following elements:

* audioFormat folder (ANSI / Windows-Cp1252)
* encoderConfig folder (UTF-8)
* logfiles folder (please do not edit)
* main.cfg configuration file (ANSI / Windows-Cp1252)
* directories.cfg (UTF-8)
* namesToCut.txt (UTF-8)
* subtitlesToCut.txt (UTF-8)
* toCode.txt (ANSI / Windows-Cp1252)

### What does Avior do?

* Process all jobs via toCode.exe or database
* Parse DVBRecService logfiles
* Compare new files against a configurable folder directory (with options to re-encode or discard)
* Search for any user-configurable terms to whitelist or blacklist an existing file for re-encoding
* Audio codec determination + re-encode based on better codec
* Video file size estimation for re-encoding (save space)
* encode files via HandbrakeCLI or ffmpeg
* Move old/existing/obsolete files to defined folders.

### What does Infuser do?

* Push jobs to a local toCode.exe file
* Push jobs to a database
* Dynamically assign jobs to specific clients via database depending on availability and load

### Workflow Example

* DVBRecService finishes recording a file and calls Infuser
* Infuser pushes job
  * to a database if database configuration was enabled depending on whether a computer is available
  * to a local toCode.txt file if database access was disabled or no eligible computers were found in the database
* Avior reads a job (toCode.txt first, then database)
* Avior parses logfiles of said recording (optional ability for old DVBRecService formats included)
* Avior tries to find a duplicate on your harddrives or network, depending on which directories you added.
* (Optional) Avior parses the video length and compares it to its actual recorded length and decides if the file is fit for processing
* (Optional) Avior parses the audio format if a duplicate was found and replaces the file if the new recording has a better audio format (stereo/multich)
* (Optional) Avior performs a file size estimation if the audio format is equal and allows re-encoding if the size is smaller by a user configurable percent amount
* (Optional) Avior checks if the logfile has blacklist or whitelist strings and decides accordingly if the old file should be replced
* (Optional) Avior automatically overwrites files if the logfile format of the duplicate is incorrect
* Avior parses the file's resolution (information present in log file of recording provided by DVBRecService)
* Avior assigns the preferred encoder (if a suitable configuration was found) to the task
* Avior cuts the movie's name and subtitle and creates an output that can be read with 90% certainty by an IMDB scraper
* Avior starts the encoder with the parameters present in the encoder configuration folder
* Avior cleans up the file from DVBRecService and moves it to an appropriate location (done/existing)


Files, Formats, Configuration
======

## toCode.txt
__This file needs to be saved in Cp1252 format if edited manually.__

This file will be generated by Infuser automatically, but I'll explain its functionality here, because its relevant for error handling.

The toCode file is a joblist containing all files to be encoded. These are automatically added by using the task feature from DVBRecording Service.

You can manually add jobs by doing so:
```
__s__
path=Drive:\Folder\filename.ts
name=Filename
sub=Subtitle
xpar=--encoderParameter
lengthOverride
__e__
```
* `__s__` and `__e__` seperate the different jobs from each other.
* `path` is the path to the source file
* `name` is the name of the video file
* `subtitle` is the 2nd part of the name of a video file
* `xpar` specifies custom encoder settings that override the encoderConfig settings
* `lengthOverride` disables the length difference module (defined in main.cfg)  

As soon as the processing begins, the job will be removed and stored in memory.

If an error occurs or a file duplicate is found, a Re-Encode string will be included to the logfiles to copy paste into toCode.txt


main.cfg:
-----

Lines starting with '#' will be ignored. The main configuration file contains all supported resolutions defined by the user as well as toggles for optional features. The editing of the main.cfg file is fully supported in the new configuration GUI. The following documentation should only serve as clarification for what each setting does.

#### Tags
The configuration file supports an unlimited amount of resolutions via tags.
You can add a configuration file for a specific resolution by giving it a name, a resolution string and an encoder to use.
A tag entry has a key, `res:` and two values that are delimited with`:`

You can add, edit and remove new tags with the GUI. If you do so, the corresponding file in the encoderConfig folder will be created. You can also specify which encoder should be used for which configuration file.

By default the GUI will suggest the preferred Encoder for a resolution.

The tags look like this:

* `res:fhd=1920x108:hb` (some TV channels send in 1920x1088, some in 1920x1080, this includes both)
* `res:hd=1280x720:ff`
* `res:tag=resolution:enc` EXAMPLE (tag is name for a resolution, resolution is the string, enc is the encoder name)

#### General configuration

* File size limit: If a recorded file is bigger than X GB, it will not be encoded
  * `gb_size x`
  * (Integer, 0 has no effect)


* Length difference: If a recorded file is shorter by x % than the estimated length suggested by DVCRecSvc, it will not be encoded
  * `ldiff_percent 10`
  * (Integer between 0 and 100, 0 has no effect)

* Process priority selection box: `priority` determines process priority of the handbrake process from 0(lowest) - 5(realtime)

* Preferred encoder sets the encoder that should be used for encoding a file. Currently two encoders are supported. The configuration key is `encoder` `<ff|hb>`
  * Handbrake (internally known as hb in the software for tag assignment and logfiles)
  * ffmpeg (internally known as ff)

* Obsolete file moving
  * `obsolete_parent_dir D:\` parent directory of the `.obsolete` folder that will be created automatically. If unspecified, the obsolete folder will be created in the source video file's directory.


**Main Tab: Intelligent Re-Encoding Module (IREM):**

| GUI Name                                   | configuration key               | value                                                                              |
|--------------------------------------------|---------------------------------|------------------------------------------------------------------------------------|
|Intelligent Re-Encode Mechanism checkbox    |`irem_enabled true/false`        |enables or disables IREM                                                            |
|IAC Accuracy slider                         |  `iac_enabled` (low, med, high) |specifies the accuracy of audio detection. More info under the IAC section          |      
|Encoding Estimaton Sample Count field       |`estimation_length_slices 2`     |specifies maximum amount of slices for a file                                       |  
|Encoding Estimation File % field            |`estimation_length_percental 5`  |defines what percentage of the total file length should be used                     |  
|File Size Difference % field                |`irem_sizediff_percent 20`       |At what percentage in file length should re-encoding be permitted                   |
|File age before re-encode in days           |`ìrem_date_disable`              |Skip re-encoding if file is younger than `days`                                     |  
|Enable legacy logfile handling checkbox     | `legacy_logfile_handling true`  | enables reading and handling of old DVBRec .mpg.log, .mkv.log  and .log file types |
|Full-text logfile search for re-encode      |`fulltext_search_existing_true  `|enables search in logfiles to find strings to help with re-encoding decision        |
|Prioritize inclusion / neutral / Prioritize exclusion slider in Logfile Full-Text Search tab  |`fulltext-search-priority 0` | can be 0, 1, 2. Determines priority for the fulltext logfile feature.                |
|Auto overwrite files using legacy logs      |`legacy_logfile_overwriting true`| automatically re-encodes all files with old logs if detected section. Note: using hyphen instead of underscores is not intentional and will be fixed soon.              |


 **Database Tab**

| GUI Name                                   | configuration key            | value            |
|--------------------------------------------|------------------------------|------------------|
| Use database checkbox |`db_enabled`|either true or false will enable or disable database support|
| Connection URL| `db_address` |contains the URI for the database server. Do NOT include a MongoDB protocol |


encoderConfig folder:
-----
This is where the encoder configuration files are placed in. They are either created by using the GUI's "Resolutions" tab or manually if desired. The GUI will create configuration files if they don't exist, but will not delete them when you remove a resolution tag. As such, you can quickly reassign resolutiosn to their corresponding configuration file by renaming them manually in this directory.

* `fhd.cfg`
* `hd.cfg`
* `tag.cfg`

The application will now look for these files and tries to read and parse its content if a matching resolution has been detected.

#### Handbrake Variables for tag.cfg:
All handbrake commands in the form of '--command' are supported. Lines starting with '#' will be ignored.

`--format mkv` __is required!__

Example:

```
--format mkv
--aencoder copy:ac3
--audio-copy-mask ac3
--audio-fallback faac
--ab 192
--encoder x265
```

#### ffmpeg variables for tag.cfg:
Accepted formats: 
* All commands starting with -
* If input parameters should be used instead, please use `PI -option`

#### File output for tag.cfg

You also have to specify an output path in this configuration file. This is the output directory of the encoded file.

* `enc_path C:\User\Videos\` (for local paths)
* `enc_path \\SERVER\Folder\` (for network paths)


Encoder selection
-----

By default, Avior will attempt to use Handbrake (configured via global System environment variable).
Handbrake is also the default fallback if for some reason ffmpeg (which is natively available) cannot be used.

As such, Handbrake is the default preferred encoder when generating a new configuration file.

## Encoder selection mechanism

If you have configured ffmpeg and handbrake for the same resolution, Avior will use the following decision logic.

* If an enocderConfig file for the preferred encoder was found, use the preferred encoder
* If an encoderConfig file for the preferred encoder was **not** found, try to find a config file for the secondary encoder

directories.cfg
-----

This file is relevant for the checking of existing files. When you add folders (one per line) to this file, the application will from now on attempt to find existing files for each job. It supports also local and network paths:

* Local paths: `D:\folder\subfolder`
* Network paths: `\\Server\folder`

File name construction & File handling
-----

Our job usually looks like this:

```
__s__
path=Drive:\Folder\I want to cut this filename - information is useful but awful new stuff 10_30_2016 .ts
name=I want to cut this filename
sub=Information is useful but awful new stuff
__e__
```

By default, name and subtitle are trimmed, concatenated and seperated by a '-' dash. In this case:

`I want to cut this filename - Information is useful but awful new stuff.mkv`

#### subtitlesToCut.txt and namesToCut.txt

These files are used to determine what strings should be cut from the name and subtitle.

Now we realize 'I want to cut this' and 'but awful' is present in a lot of files we process. In reality, this often applies to strings such as "Movie", "Documentary", "Fernsehfilm", "KinoFestival" or similar that are added to the name or subtitle.
All we need to do is add a new line to namesToCut.txt containing `I want to cut this`.
The same applies for subtitlesToCut.txt with `but awful`

The behavior for these two options are different:
* Strings found in namesToCut are cut __at the beginning__ of the name string
* Strings foudn in  subtitlesToCut are cut __at the first occurence__ and removes **everything behind it**

So in our case, that would be `I want to cut this filename` being corrected to `filename` and `Information is useful but awful` being corrected to `Information is useful`

Our filename will then be: `filename - Information is useful.mkv`

Default behavior for existing files:
-----

Every source file (not encoded) with IREM disabled will be moved to an `existing` folder relative to the source file's path.
With IREM enabled, see the documentation below. If IREM prohibits re-encoding, the default behavior will be used.

Default behavior for encoded files
-----

Every source file of a finished job will be moved into a `done` directory relative its path


Intelligent Re-Encode Mechanism:
-----
The IREM module can be activated by setting the `irem_enabled` flag to `true` in the `config.cfg` file.

If a video file that is to be encoded is already found, it will not automatically
be moved to the `existing` folder, but checked for a number of properties.
These checks can be manipulated via user configurable parameters to accommodate to users' needs.



Currently there are 4 modules available for IREM that can potentially help on making a re-encode decision

* legacy logfile fulltext search
* logfile fulltext
* audio format check (Intelligent Audio Comparator)
* file size estimation

### Logfile fulltext search

This module has two opertion modes.
Handling legacy logfiles and handling normal logfiles.

* If legacy logfile handling has been enabled in the main configuration and a detection takes place,
only if the result is positive a re-encoding will be allowed.
* If a regular logfile is encountered and the result is negative,
all other activated IREM modules will be used subsequently.

##### How it works
By default, there are two files available in the `searchTerms` folder, `searchTermsInclude.txt` and `searchTermsExclude.txt`, where each line represents one search term. Again, a full GUI implementation is available and recommended to use. It can be found in the main config section under "Logfile Full-Text search"

* Inclusion means that if a match is found the file will be re-encoded
* Exclusion means that if a match is found the file will __not__ be re-encoded.
* `fulltext-search-priority` parameter in the main configuration determines what to prioritize. Choosing a value of 0 will prioritize inclusion, 1 is neutral which does not re-encode a file if both inclusion and exclusion lists find a match, 2 will prioritize exclusion.

##### Supported logfile types
Regular mode:
* .txt
* .log

Legacy mode:
* .mkv.log
* .mpg.log
* .log (without paired .txt)


### Intelligent Audio Comparator

__IAC is a part of IREM since avior_001__

#### Algorithm

Here's a table of how the Intelligent Audio Comparator (IAC) algorithms decides.

| Audio format                                     | Value         |
|:-------------------------------------------------|:-------------:|
| None in log                                      | 0             |
| Stereo only in log                               | -3            |
| Multich only in log                              | +3            |
| Multi + Stereo in log, Stereo tag in Txt         | -2            |
| Multi + Stereo in log, No tag                    | +1            |
| Multi + Stereo in log, Multi tag in Txt          | +2            |

  The value determines accuracy. Positive values mean it's more likely to be multichannel, negative is the same for stereo.
  * If the .log file contains only __one__ type of audio format, the audio format information is always 100% accurate and will be chosen at all times. The .txt file will be ignored.  The assigned values for Stereo and Multichannel are -3 and +3.

  * If the .log file contains two audio formats __and__ there are tag information available in the .txt file, those will be chosen over the .log information. This is accurate as long as the TV channels provide valid tags.
  The assinged values for Stereo and Multichannel are -2 and +2

  * If the .log file contains two audio formats and there are __no tags__ available in the .txt file, it will always be treated as multichannel file, since the probability is higher than being a stereo file. The assigned value for this case is +1

#### audioFormat Folder

Within the audioFormat folder you can find two files:
* multichannel.cfg
* stereo.cfg

You should write the audio formats from the filename.log and filename.txt logfiles provided by DVBRecService into these configuration files.

Sample filename.log:
```
23:51:13 / 00:00:00 (~ 0,00 MB) Start
23:51:13 / 00:00:00 (~ 0,25 MB) PID 6310: H.264 Video, 16:9, 480x360, 50 fps
23:51:13 / 00:00:00 (~ 0,25 MB) PID 6322: AC3 Audio 5.1, 48 khz, 448 kbps
01:54:38 / 01:33:25 (~ 0,25 MB) PID 6322: AC3 Audio Stereo, 48 khz, 448 kbps
01:24:39 / 01:33:26 (~ 7124,39 MB) Stop
```

Sample filename.txt

```
Id=6930
Date=07.03.2016
Time=20:15:00
Duration=01:35:00
Description=Random Description [stereo] [AC-3] [fra] [de]
```

Now you should include to the multichannel.cfg file:
* `AC3 Audio 5.1`
* `[AC-3]`

And you should include these to the stereo.cfg file:
* `AC3 Audio Stereo`
* `[stereo]`

If you delete the audioFormats folder, a new set with preconfigured values will be generated the next time a file is processed.

### Size estimation

* The estimation algorithm will encode x amount of y minute slices. y is determined by using the file size difference field in he configuration
* With `estimation_length_percental` you can define how big the fraction of the file that will be encoded is.
Example: setting it to `10` on a 100 minute video file will encode 10 slices@1min length
* With `estimation_slices` you can define the number of samples that should be taken.
With the aforementioned example, setting this value to `3` will only encode 2 slices @ y min.
* Slices will be added up and an average Megabyte per minute value is calculated to be used in estimation of the encoded video file total size.
* `irem_sizediff_percent` specifies how big the file size difference has to be to permit encoding. Setting it to `10` will permit encoding if the new file is estimated to be 900MB and the old was 1000MB.
* Progress will be shown in the UI

##### Calculation of maximum Length
```Java
int maxLength;
if (conf.getEstimationLengthMaxMinutes() > ts.length) {   //ts.getLength is the video file length in minutes
    maxLength = ts.length;
} else {
    maxLength = conf.getEstimationLengthMaxMinutes();
}
int count = (int)Math.ceil((ts.length * (conf.getEstimationLengthPercental() * 0.01)));
    if (count > maxLength) {
      count = maxLength;
    }
```

##### Calculation and positioning of re-encode slices
```Java
        //Time units is how many seconds slices are apart from each other
        int timeUnits = (ts.getLength() *60) / (slices + 1);
        //slices + 1 because there needs to be room at the end of the file and the entry point mustn't be EOF for 1 slice.
        int totalDuration = 0;
        ArrayList<Double> sampleSizes = new ArrayList<>();
        //Start at half the time unit to avoid hitting opening sequences / black screens in movies
        int position = timeUnits / 2;
        //If encodingDuration in minutes is greater or equal to size, encode the whole thing
        int encodingDurationMinutes = encodingDuration / 60;
        if (encodingDurationMinutes >= ts.getLength()) {
            slices = 1;
            position = 0;
            timeUnits = 0;
            encodingDuration = ts.getLength() * 60;
        }
```

* `timeUnits` is the value in seconds how far the slices are timed in apart
* `position` is the position in seconds of the video file. The start is half the time units
    * Example: Video file length is 100 minutes, percental is 10, slices is 10. Then timeUnits equals 600 seconds.
    * We will then get an encode at the following positions:
      1. 300 - 360 seconds (position 300)
      2. 900 - 660 seconds (position 900)
      3. 1500 - 1560 seconds
      4. 2100 - 2160 seconds
      5. 2700 - 2760 seconds
      6. 3300 - 3360 seconds
      7. 3900 - 3960 seconds
      8. 4000 - 4060 seconds
      9. 4600 - 4660 seconds
      10. 5200 - 5260 seconds
    * Total video file length is 6000 seconds, so we have a relatively even-spreaded measurement


#### Example of how IREM looks like in the log file for an error case:
```
starting IREM algorithm...done. Estimation Percental/MaxLength: 3%/6min, Size Threshold: 20 AudioFormat Detection Acc: 1
analyzing audio for: I:\Recording\Candice Renoir_2017-03-13-10-28-16-zdf_neo HD (AC3,ger) successful: Stereo (-3)
analyzing audio for: \\VDR\SerienHD\Candice Renoir\Candice Renoir - Der Mensch ist dem Menschen ein Wolf successful: Stereo (-3)
comparing audio formats... equal. Moving to file size comparison

encoding 2 slices@60 seconds for estimation... done, time elapsed: 10 min., avg slice size: 13.0 MB
comparing file size... done. Encoding forbidden. New file (702.0 MB) larger by 30% compared to old file (540.901908 MB)
fallback to default existing behavior... done
moving files to \existing\ directory... completed
```

Logfiles
-----

DvB Recording Service by default provides two kinds of logfiles: file.log and file.txt.
VDRAvior will use the .log file for appending encoding information, the .txt file stays unmodified, but the modified date will be changed for easier file explorer sorting.

The built in Logger is one of the most powerful modules in this application.

* Within the __logfiles__ folder you can find an error log and a success log.
  * The __error log__ will contain malfunction reports as well as information about why a file was not encoded. This depends on the modules you have activated and the configuration.
  * The __success log__ contains information about successfully encoded files.
* In addition, the Logger will create a `filename.ts.RESULT.LOG:`file at the location of the source file (the raw file outputted by DVBRecService) to let the user know whether there was an error. You will receive an __automatically generated re-encode template__ suited for copy paste to the `toCode.txt` file
* If encoding was successful, information about the encoding duration and audio format (if IAC module was enabled) will be appended to the DVB Recording Service .log file.

Example full logfile in an error case due to insufficient file size decrease in an IREM re-encode case:
```
----------------
Mon Mar 13 11:26:20 CET 2017

processing file: I:\Recording\Candice Renoir_2017-03-13-10-28-16-zdf_neo HD (AC3,ger).ts
checking file size... ok
reading logfiles... completed.
reading file resolution... completed: hd (ff)

parsing recorded length... completed: 54 minutes
parsing expected length... completed: 55 minutes
checking video length... ok
loading encoder config... successful

searching for existing files... found
Path: \\VDR\SerienHD\Candice Renoir\Candice Renoir - Der Mensch ist dem Menschen ein Wolf.mkv
Size: 540 MB
reading first duplicate's logfiles... completed.

starting IREM algorithm...done. Estimation Percental/MaxLength: 3%/6min, Size Threshold: 20 AudioFormat Detection Acc: 1
starting IREM algorithm...done. Estimation Percental/MaxLength: 3%/7min, Size Threshold: 10, AudioFormat Detection Acc: 1
launching fulltext search for logfiles... done. Running in regular inclusion mode
sifting through .log file... match found: And then he transcended.
launching fulltext search for logfiles... done. Running in regular exclusion mode
sifting through .log file... no match
sifting through .txt file... match found: Filmdebut im Ersten
applying search priority (1) to results... done. full-text search brought forth no results. (no-priority-mutex) Moving to audio format comparison

analyzing audio for: I:\Recording\Candice Renoir_2017-03-13-10-28-16-zdf_neo HD (AC3,ger) successful: Stereo (-3)
analyzing audio for: \\VDR\SerienHD\Candice Renoir\Candice Renoir - Der Mensch ist dem Menschen ein Wolf successful: Stereo (-3)
comparing audio formats... equal. Moving to file size comparison

encoding 2 slices@60 seconds for estimation... done, time elapsed: 10 min., avg slice size: 13.0 MB
comparing file size... done. Encoding forbidden. New file (702.0 MB) larger by 30% compared to old file (540.901908 MB)
fallback to default existing behavior... done
moving files to \existing\ directory... completed

Re-Encode Template:
__s__
path=I:\Recording\existing\Candice Renoir_2017-03-13-10-28-16-zdf_neo HD (AC3,ger).ts
name=Candice Renoir
sub=Der Mensch ist dem Menschen ein Wolf
lengthOverride
__e__

Operation FAILED
```
Database with Infuser and Avior
======

## Avior database client management

A client object has the following properties

* `Name` is the client name
* `Available from` is the time when Avior will process jobs
* `Available to` is the time when Avior will stop processing jobs
* `Maximum jobs` is the maximum amount of jobs that can be pushed to one client
* `Priority` is the priority used to determine which client should be used for a job
* `Online` is a non editable field that shows whether the client is currently registerd or not

With the available fields a schedule can be set if a computer should not encode jobs 24/7.

The infuser will by default not use a database. Instead it will put every job in the local toCode.txt file.

However if a database string is specified in the InfuserDB.conf, its behavior will change significantly.

## How Infuser selects an eligible client for a job

By default, if Avior is configured with a database it will create a client entry for itself with default values. Those can be seen and edited in the Avior Main Config GUI under the Database tab.

whenever Avior is started with database support enabled, it will register itself with the database and set its online field to true.

At this point, Infuser will check if a client is eligible for receiving a job. It'll use the following criteria:

* if a client is online, it will be considered for a job
* If a client hasn't reached its maximum jobs number, it will be considered for a job
* The client with the highest priority and free jobs will receive a job
* If the priority is equal and both haven't reached their maximum, Infuser will try to evenly spread out the jobs
* If no client is online and eligible but the same computer from which Infuser was called has a DB entry, it will receive the job
* If no client is online and eligible and the computer from which Infuser was called does not have a DB entry the job will be written to the local toCode.txt file. This is a last resort mechanism to prevent jobs from getting lost
