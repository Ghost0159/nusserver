# Custom Nintendo Update Server

This is my implementation of a custom Nintendo Update Server (NUS) in Python. Started in May or September 2017.

Currently supports 3DS (Old+New) and WiiU (+vWii). Other platforms may be supported if a way to use a custom server can be done.

Only tested with Python 3.6, but it should work with at least 3.5.

This does not handle serving title contents. Another server must serve the files. A quick method (but not recommended beyond testing) is: `python3 -m http.server`

Most of this is made after poking around with Fiddler for a few hours, but I would suggest reading [Yifan Lu's post about the 3DS system updater](https://yifan.lu/2015/03/23/nintendo-3ds-system-updater/) for more details about the process.

This is far from finished and some bad assumptions are probably made. It works when I used it though!

## Usage / Configuration
Install `lxml` using pip.

Copy `nusconfig.py.template` to `nusconfig.py` and edit the values described below:

* `port` - Port for the server to listen on.
* `address` - Address to access the server.
* `content_prefix_url` - URL to point to for title downloads, depending on console and region. 3DS/WiiU downloads title contents with this.
  * Defaults:
    * 3DS: `http://nus.cdn.c.shop.nintendowifi.net/ccs/download`
    * WiiU: `http://nus.cdn.wup.shop.nintendo.net/ccs/download`
* `uncached_content_prefix_url` URL to point to for "uncached" title downloads, depending on console and region. 3DS/WiiU downloads title tmds with this. This can be the same as content_prefix_url.
  * Defaults:
    * 3DS: `https://ccs.c.shop.nintendowifi.net/ccs/download`
    * WiiU: `https://ccs.wup.shop.nintendo.net/ccs/download`
* `cdn_directory` - Path to directory with the cdn contents, depending on console and region. This is only used to read tickets from when the system requests for tickets. Currently it must be layed out like cdn (i.e. `/your/path/<titleid>/cetk`), this should change in the future to something simpler.
* `titlehash` - Titlehash to respond to, depending on console and region. `RANDOM` will generate and save a random titlehash, then always send it when the system requests it. See "Note about titlehashes" below for more information.

| Codename | Console name |
| --- | --- |
| `ctr` | Old3DS |
| `ktr` | New3DS |
| `wup` | WiiU |
| `wupv` | WiiU vWii |
| `rvl` | Wii (NYI) |
| `twl` | DSi (NYI) |

### Certs
Ticket certs must be set up. `get-certs.py` can get these for 3DS, WiiU, and WiiU vWii. Requires the [requests](http://docs.python-requests.org/) module.

### Setting up a title list
The title list is sent during the update process. This custom server reads it from a csv in the `/tidlist` directory. The format is a simple `titleid,version` layout. The filename format is `<codename>-<region>.csv`. For example, a USA Old3DS would use `/tidlist/ctr-usa.csv`.

There are some scripts in the `tools/` directory to generate a tidlist:
* `gen-tidlist-from-cias.py` - Generate a tidlist from a directory containing CIA files.
  * Example: `gen-tidlist-from-cias.py updates > tidlist/ctr-usa.csv`
* `gen-tidlist-from-ninupdates.py` - Generate a tidlist from a ninupdates csv file.
  * Example: `gen-tidlist-from-ninupdates.py 8.1.0-19U.csv usa > tidlist/ctr-usa.csv`
* `gen-tidlist-from-ninupdatesurl.py` - Generate a tidlist from a ninupdates url. Requires the [requests](http://docs.python-requests.org/) module.
  * Example for Old 8.1.0-19U: `gen-tidlist-from-ninupdatesurl.py https://yls8.mtheall.com/ninupdates/reports.php?date=08-07-14_02-05-03&sys=ctr usa > tidlist/ctr-usa.csv`

## Using on console
### 3DS
Two ways of redirecting to a custom domain:
* Patch SSL to disable Root CA verification, then redirect with a tool like Fiddler: https://github.com/SciresM/3DS-SSL-Patch
* Patch NIM to redirect two urls (custom url must be same length or shorter):
  * `https://ecs.c.shop.nintendowifi.net/ecs/services/ECommerceSOAP` -> `http://your-server/ecs/services/ECommerceSOAP` (appears twice)
  * `https://nus.c.shop.nintendowifi.net/nus/services/NetUpdateSOAP` -> `http://your-server/nus/services/NetUpdateSOAP`
  * Replacing the ECommerceSOAP url will break eShop since it is also used for other purposes. I would guess one is used for updates, and the other for eShop, but I didn't try this yet.

The 3DS does not handle HTTP redirects. Any redirect given will result in an error.

#### Note about titlehashes
The titlehash is a quick way for systems to determine if there is a new system update without comparing every installed title to a list returned from the server. The first request in the update process is for the titlehash. If it is identical to the saved hash, the process immediately ends.

For 3DS (maybe others), a request for the titlehash is also made after downloading all titles to update (sometimes it is requested before all titles are downloaded). If the hash is different, the update fails and nothing is installed.

This is why always sending a random hash won't work, so saving a random value and always sending it means the system can actually finish the update. This custom server allows resetting random hashes to another random value via a web interface. The page can be accessed at `http://your-server/`.

### Wii U
Since there is no SSL patch, [replacing a Root CA](https://www.reddit.com/r/WiiUHacks/comments/6zfck3/guide_setting_up_mitm_to_log_and_preserve_services/) would be needed for redirection with Fiddler. This should not be done without a reliable way to restore the original root cert, such as CBHC.

An unknown request is (sometimes?) made to `idbe-wup.cdn.nintendo.net` during title downloads (Update/eShop). Replacing the default Root CA doesn't work because this uses a different one ("Nintendo Class 2 CA - G3"). Based on [WiiUBrew](http://wiiubrew.org/wiki/Nn_idbe.rpl), it may not be used for anything useful here, so disabling decryption of it should be fine.

vWii is updated in Wii U mode, but uses its own title list and titlehash (determined by an additional field `nus:VirtualDeviceType` in the XML requests).

If you use the common RedNAND implementation, updates will not be installed in RedNAND, likely because OSv255 (System Updater OS) is not patched to redirect to RedNAND.

## License / Credits
`nuscommon.py`, `nusserver.py`, `tools/` are under the MIT license.
