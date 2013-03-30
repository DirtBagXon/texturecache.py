texturecache.py
===============

Utility to manage and update the local XBMC texture cache (Texture##.db and Thumbnails folder), view content of the XBMC media library using JSON calls (so works with MySQL too), and also cross reference cache with library to identify space saving opportunities or problems.

##Summary of features

*Typically a lower case option is harmless, an uppercase version may delete/modify data*

**[c, C]** Automatically re-cache missing artwork, with option to force download of existing artwork (remove first, then re-cache). Can use multiple threads (default is 2)

**[nc]** Identify those items that require caching (and would be cached by **c** option)

**[p, P]** Prune texture cache by removing accumulated **cruft** such as image previews, previously deleted movies/tv shows/music and whose artwork remains in the texture cache even after cleaning the media library database. Essentially, remove any cached file that is no longer associated with an entry in the media library, or an addon

**[s, S]** Search texture cache for specific files and view database content, can help explain reason for incorrect artwork. Option **S** returns only those items that no longer have a matching image in the file system.

**[x, X]** Extract rows from texture cache database, with optional SQL filter. Can also be used to return only those database rows referencing a cached image that no longer exists in the file system

**[d]** Delete specific database rows and corresponding files from the texture cache using database row identifier (see **s/S**)

**[r, R]** Reverse query cache, identifying any "orphaned" files no longer referenced by texture cache database, with option to auto-delete files

**[j, J, jd, Jd]** Query media library using JSON API, and output content using JSON notation (and suitable for further external processing). The **jd** and **Jd** options will decode (unquote) all artwork urls. **J** and **Jd** options will include additional user configurable fields when querying media library (see [properties file](#optional-properties-file))

**[qa]** Perform QA check on media library items, identifying missing properties (eg. plot, mpaa certificate, artwork etc.). Default QA period is previous 30 days, configurable with [qaperiod](#optional-properties-file). Add properties "[qa.rating = yes](#optional-properties-file)" or "[qa.file = yes](#optional-properties-file)" for rating and file validation during QA

**[qax]** Like the **qa **option, but also performs a library remove and then library rescan of any media folder found to contain media items that fail a QA test

##Installation instructions

Download the single Python file required from github. A default properties file is available on github, rename this to texturecache.cfg in order to use it.

To download the script at the command line:
####Code:
```
wget https://raw.github.com/MilhouseVH/texturecache.py/master/texturecache.py -O texturecache.py
chmod +x ./texturecache.py
```

If you are using OpenELEC which has a pretty basic wget that doesn't support https downloads, instead use `curl`:

####Code:
```
curl https://raw.github.com/MilhouseVH/texturecache.py/master/texturecache.py -o texturecache.py
chmod +x ./texturecache.py
```

##Basic Example usage
Let's say the poster image for the "Dr. No" movie is corrupted, and it needs to be deleted so that XBMC will automatically re-cache it (hopefully correctly) next time it is displayed:

1) Execute: `./texturecache.py s "Dr. No"` to search for my Dr. No related artwork

2) Several rows should be returned from the datbase, relating to different cached artwork - one row will be for the poster, the other fanart, and there may also be rows for other image types too (logo, clearart etc.). This is what we get for Dr. No:

```
000226|5/596edd13.jpg|0720|1280|0011|2013-03-05 02:07:40|2013-03-04 21:27:37|nfs://192.168.0.3/mnt/share/media/Video/Movies/James Bond/Dr. No (1962)[DVDRip]-fanart.jpg
000227|6/6f3d0d94.jpg|0512|0364|0003|2013-03-05 02:07:40|2013-03-04 22:26:38|nfs://192.168.0.3/mnt/share/media/Video/Movies/James Bond/Dr. No (1962)[DVDRip].tbn
Matching row ids: 226 227
```

3) Since only the poster (.tbn) needs to be removed, executing `./texturecache.py d 227` will remove both the database row *and* the cached poster image. If we wanted to remove both images, we would simply execute `./texturecache.py d 226 227` and the two rows and their corresponding cached images would be removed.

Now it's simply a matter of browsing the Dr. No movie in the XBMC GUI, and the image should be re-cached correctly.

But wait, there's more... another method is to force images to be re-cached, automatically. `./texturecache.py C movies "Dr. No"` will achieve the same result as the above three steps, including re-caching the deleted items so that it is already there for you in the GUI.

##Media Classes

The utility has several options that operate on media library items grouped into classes:

* albums
* artists
* songs
* movies
* sets
* tags
* tvshows

In most cases, when performing an operation it is possible to specify a filter to further restrict processing/selection of particular items, for example, to extract the default media library details for all movies whose name contains "zombie":
####Code:
```
./texturecache.py j movies zombie
```

##Tag Support
When using the tags media class, you can apply a filter that uses logical operators such as `and` and `or` to restrict the selection criteria.

For example, to cache only those movies tagged with either action and adventure:
####Code:
```
./texturecache.py c tags "action and adventure"
```

Or, only those movies tagged with either comedy or family:
####Code:
```
./texturecache.py c tags "comedy or family"
```

If no filter is specified, all movies with a tag will be selected.


##Format of database records

When displaying rows from the texture cache database, the following fields (columns) are shown:
####Code:
```
rowid, cachedurl, height, width, usecount, lastusetime, lasthashcheck, url
```

##Additional usage examples

#####Caching all of the artwork for your TV Shows
####Code:
```
./texturecache.py c tvshows
```

#####Viewing your most recently accessed artwork
####Code:
```
./texturecache.py x | sort -t"|" -k6
```
or
####Code:
```
./texturecache.py x "order by lastusetime asc"
```

#####Viewing your Top 10 accessed artwork
####Code:
```
./texturecache.py x | sort -t"|" -k5r | head -10
```
or
####Code:
```
./texturecache.py x "order by usecount desc" 2>/dev/null | head -10
```

#####Identifying cached artwork for deletion

Use texturecache.py to identify artwork for deletion, then cutting and pasting the matched ids into the "d" option or via a script:

For example, to delete those small remote thumbnails you might have viewed when downloading artwork (and which still clutter up your cache):
####Code:
```
./texturecache.py s "size=thumb"
```

*then cut & paste the ids as an argument to `./texturecache.py d id [id id]`

and the same, but automatically:
####Code:
```
IDS=$(./texturecache.py s "size=thumb" 2>&1 1>/dev/null | cut -b19-)
[ -n "$IDS" ] && ./texturecache.py d $IDS
```

Or when removing artwork that is no longer needed, simply let texturecache.py work it all out:
####Code:
```./texturecache.py P```

#####Delete artwork that has not been accessed after a particular date
####Code:
```
./texturecache.py x "where lastusetime <= '2013-03-05'
```

or hasn't been accessed more than once:
####Code:
```
./texturecache.py x "where usecount <= 1"
```

#####Query the media library, returning JSON results

First, let's see the default fields for a particular media class (movies), filtered for a specific item (avatar):
####Code:
```
./texturecache.py j movies "avatar"
```
####Result:
```
[
  {
    "movieid": 22,
    "title": "Avatar",
    "art": {
      "fanart": "image://nfs%3a%2f%2f192.168.0.3%2fmnt%2fshare%2fmedia%2fVideo%2fMovies%2fAvatar%20(2009)?%5bDVDRip%5d-fanart.jpg/",
      "discart": "image://http%3a%2f%2fassets.fanart.tv%2ffanart%2fmovies%2f19995%2fmoviedisc%2favatar-4eea31049147b.png/",
      "poster": "image://nfs%3a%2f%2f192.168.0.3%2fmnt%2fshare%2fmedia%2fVideo%2fMovies%2fAvatar%20(2009)?%5bDVDRip%5d.tbn/",
      "clearart": "image://http%3a%2f%2fassets.fanart.tv%2ffanart%2fmovies%2f19995%2fmovieart%2favatar-4f803992128b8.png/",
      "clearlogo": "image://http%3a%2f%2fassets.fanart.tv%2ffanart%2fmovies%2f19995%2fhdmovielogo%2favatar-503e0262ba196.png/"
    },
    "label": "Avatar"
  }
]
```

With `extrajson.movies = trailer, streamdetails, file` in the properties file, here is the same query but now returning the extra fields too:
####Code:
```
./texturecache.py J movies "Avatar"
```
####Result:
```
[
  {
    "movieid": 22,
    "title": "Avatar",
    "label": "Avatar",
    "file": "nfs://192.168.0.3/mnt/share/media/Video/Movies/Avatar (2009)[DVDRip].m4v",
    "art": {
      "fanart": "image://nfs%3a%2f%2f192.168.0.3%2fmnt%2fshare%2fmedia%2fVideo%2fMovies%2fAvatar%20(2009)?%5bDVDRip%5d-fanart.jpg/",
      "discart": "image://http%3a%2f%2fassets.fanart.tv%2ffanart%2fmovies%2f19995%2fmoviedisc%2favatar-4eea31049147b.png/",
      "poster": "image://nfs%3a%2f%2f192.168.0.3%2fmnt%2fshare%2fmedia%2fVideo%2fMovies%2fAvatar%20(2009)?%5bDVDRip%5d.tbn/",
      "clearart": "image://http%3a%2f%2fassets.fanart.tv%2ffanart%2fmovies%2f19995%2fmovieart%2favatar-4f803992128b8.png/",
      "clearlogo": "image://http%3a%2f%2fassets.fanart.tv%2ffanart%2fmovies%2f19995%2fhdmovielogo%2favatar-503e0262ba196.png/"
    },
    "trailer": "",
    "streamdetails": {
      "video": [
        {
          "duration": 9305,
          "width": 720,
          "codec": "avc1",
          "aspect": 1.7779999971389771,
          "height": 576
        }
      ],
      "audio": [
        {
          "channels": 6,
          "codec": "aac",
          "language": "eng"
        }
      ],
      "subtitle": []
    }
  }
]
```

##Optional Properties File

By default the script will run fine on distributions where the `.xbmc/userdata` folder is within the users Home folder (ie. `userdata=~/.xbmc/userdata`). To override this default, specify a properties file with a different value for the `userdata` property.

The properties file should be called `texturecache.cfg`, and will be looked for in the current working directory, then in the same directory as the texturecache.py script. What follows is an example properties file showing the default values:

```
sep = |
userdata = ~/.xbmc/userdata
dbfile = Database/Textures13.db
thumbnails = Thumbnails
xbmc.host = localhost
webserver.port = 8080
webserver.username =
webserver.password =
rpc.port = 9090
download.threads = 2
extrajson.albums =
extrajson.artists =
extrajson.songs =
extrajson.movies =
extrajson.sets =
extrajson.tvshows.tvshow =
extrajson.tvshows.season =
extrajson.tvshows.episode =
qaperiod = 30
qa.rating = no
qa.file = no
cache.castthumb = no
cache.ignore.types = image://video, image://music
logfile =
```

The `dbfile` and `thumbbnails` properties represent folders that are normally relative to the `userdata` property, however full paths can be specified.

Set values for `webserver.username` and `webserver.password` if you require webserver authentication.

The `extrajson.*` properties allow the specification of additional JSON audio/video fields to be returned by the J/Jd query options. See the XBMC [JSON-RPC API Specification](http://wiki.xbmc.org/index.php?title=JSON-RPC_API/v6) for details.

Cast thumbnails will not be cached by default, so specify `cache.castthumb = yes` if you require cast artwork to be re-cached.

Ignore specific URLs when pre-loading the cache (c/C/nc options), by specifying comma delimited regex patterns for the `cache.ignore.types` property. Default values are `image://video` and `image://music`. Set to none (no argument) to process all URLs. Any URL that matches one of the ignore types will not be considered for re-caching (and will be counted as "ignored").

Specify a filename for the `logfile` property, to log detailed processing information. Prefix the filename with + to force flushing.

Run the script without arguments for basic usage, and with `config` parameter to view current configuration information.

See texturecache.py @ XBMC Forums: [Click here to goto Forum](http://forum.xbmc.org/showthread.php?tid=158373)
=====
