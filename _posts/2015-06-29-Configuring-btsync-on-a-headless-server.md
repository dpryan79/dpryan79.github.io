---
layout: post
title: Configuring btsync on a headless server
---

We often need to distribute large sequencing datasets to remote users/institutes. While this sort of thing is typically done by setting up an FTP server, getting approval to do so (and getting ports opened in the firewall) is problematic. Consequently, we're starting to use [btsync](https://www.getsync.com/), which allows encrypted point-to-point transfers and can (largely) bypass our firewall. The only trick to this is that most of our servers that have access to sequencing data are headless and firewalled themselves, so one can't simply connect to the web UI opened by `btsync`. What follows is how I go about configuring btsync purely from the command line.

Configuration File
==================

`btsync` uses a configuration file if you specify the `-c` option. An example file can be produced with `./btsync --dump-sample-config`:

    {
      "device_name": "My Sync Device",
      "listening_port" : 0, // 0 - randomize port
    
    /* storage_path dir contains auxilliary app files if no storage_path field: .sync dir created in the directory
       where binary is located. otherwise user-defined directory will be used */
    // "storage_path" : "/home/user/.sync",
    
    /* set location of pid file */
    // "pid_file" : "/var/run/btsync/btsync.pid",
    
    /* use UPnP for port mapping */
      "use_upnp" : true,
    
    /* limits in kB/s. 0 - no limit */
      "download_limit" : 0,
      "upload_limit" : 0,
    
    /* proxy configuration */
    // "proxy_type" : "socks4", // Valid types: "socks4", "socks5", "http_connect". Any other value means no proxy
    // "proxy_addr" : "192.168.1.2", // IP address of proxy server.
    // "proxy_port" : 1080,
    // "proxy_auth" : false, // Use authentication for proxy. Note: only username/password for socks5 (RFC 1929) is supported, and it is not really secure
    // "proxy_username" : "user",
    // "proxy_password" : "password",

      "webui" :
      {
        "listen" : "0.0.0.0:8888" // remove field to disable WebUI
    
    /* preset credentials. Use password or password_hash */
    //  ,"login" : "admin"
    //  ,"password" : "password"
    //  ,"password_hash" : "some_hash" // password hash in crypt(3) format
    //  ,"allow_empty_password" : false // Defaults to true
    /* ssl configuration */
    //  ,"force_https" : true // disable http
    //  ,"ssl_certificate" : "/path/to/cert.pem"
    //  ,"ssl_private_key" : "/path/to/private.key"
    
    /* directory_root path defines where the WebUI Folder browser starts (linux only). Default value is / */
    //  ,"directory_root" : "/home/user/MySharedFolders/"
    
    /* directory_root_policy defines how directory_root is used (linux only).
       Valid values are:
         "all" - accepts directory_root and its subdirectories for 'getdir' and 'adddir' actions
         "belowroot" - accepts directory_root's subdirectories for 'getdir' and 'adddir' actions,
         but denies attempts to use 'adddir' to create directories directly within directory_root
       Default value is "all". */
    //  ,"directory_root_policy" : "all"
    
    /* dir_whitelist defines which directories can be shown to user or have folders added (linux only)
       relative paths are relative to directory_root setting */
    //  ,"dir_whitelist" : [ "/home/user/MySharedFolders/personal", "work" ]
      }
    
    /* !!! if you set shared folders in config file WebUI will be DISABLED !!!
       shared directories specified in config file  override the folders previously added from WebUI. */
    /*,
      "shared_folders" :
      [
        {
          "secret" : "MY_SECRET_1", // required field - use --generate-secret in command line to create new secret
          "dir" : "/home/user/bittorrent/sync_test", // * required field
          "use_relay_server" : true, //  use relay server when direct connection fails
          "use_tracker" : true,
          "use_dht" : false,
          "search_lan" : true,
          "use_sync_trash" : true, // enable SyncArchive to store files deleted on remote devices
          "overwrite_changes" : false, // restore modified files to original version, ONLY for Read-Only folders
          "known_hosts" : // specify hosts to attempt connection without additional search
          [
            "192.168.1.2:44444"
          ]
        }
      ]
    */
    
    /* Advanced preferences can be added to config file. Info is available at http://sync-help.bittorrent.com */
    
    }

Much of that is actually not needed in our case, since we very much do not want to use the web UI. The really important parts are:

 * `device_name` : This can be anything, though it's better if it's something that's meaningful to you.
 * `storage_path`: This is a temp. directory, most conveniently under ~/.sync.
 * `pid_file`: I typically put this under the `storage_path`
 * `use_upnp`: Can be `true` or `false`. Our firewall blocks upnp, so I use `false`.
 * The `shared_folders` section, described below.

Aside from the `shared_folders` section, the beginning of my configuration file looks like the following:

    {
      "device_name": "MPI-IE Sequence Sharing",
      "listening_port" : 0, // 0 - randomize port
      "storage_path" : "/home/ryan/.sync",
      "pid_file" : "/home/ryan/.sync/btsync.pid",
      "use_upnp" : false,
      "download_limit" : 0,
      "upload_limit" : 0,

The `shared_folders` section specifies the actual folders that are to be synchronized and their associated keys. We typically want to do a unidirectional sync, where changes made by remote users do nothing on our local server (otherwise, users could accidentally delete the original files!). This part is done with a pair of keys. The first key allows read/write access and should be used on the server holding the original files. That key can then be used to generate the read-only key, which can then be used by remote users.

Key generation
--------------

We can generate a pair of keys as follows:

    btsync --generate-secret
    btsync --get-ro-secret returnedKey

If the first command returned `AHEB3VJFBSYHKQDRMPJFCTIWQ3FRFGV2Z`, then we would generate the read-only key with `btsync --get-ro-secret AHEB3VJFBSYHKQDRMPJFCTIWQ3FRFGV2Z`, which would produce `B5C5PQMWPUYT7GUG473AY6MCHB6LJVVTI`.

I'll use these in the rest of the examples.

Master configuration
--------------------

The `shared_folders` section of the master and remote computers can be nearly identical, excepting the key. Let's suppose that I wanted to share /home/ryan/Downloads/test. This section in the config file could then look like:

    "shared_folders" :
      [
        {
          "secret" : "AHEB3VJFBSYHKQDRMPJFCTIWQ3FRFGV2Z",
          "dir" : "/home/ryan/Downloads/test",
          "use_relay_server" : true, 
          "use_tracker" : true,
          "use_dht" : false,
          "search_lan" : false,
          "use_sync_trash" : false,
          "overwrite_changes" : false,
        }
      ]

Remote configuration file
-------------------------

Given the last section, suppose I want to sync the above directory to `/home/ryan/Downloads/test2` on a different machine. My configuration file would then look like:

    "shared_folders" :
      [
        {
          "secret" : "B5C5PQMWPUYT7GUG473AY6MCHB6LJVVTI",
          "dir" : "/home/ryan/Downloads/test2",
          "use_relay_server" : true, 
          "use_tracker" : true,
          "use_dht" : false,
          "search_lan" : false,
          "use_sync_trash" : false,
          "overwrite_changes" : false,
        }
      ]

Note that the only differences are the `secret` and the `dir` lines.

Misc. notes
-----------

The directories specified by `dir` and `storage_path` must exist (`storage_path` does not need to exist on read-only computers). Otherwise, you will get the following error:

    Storage path specified in config file does not exist.

It can be non-trivial to know when a synchronization is actually complete, since without the UI there is no visual feedback. You can, however, look in the log file specified under `storage_path`. If you see a line with `Disconnect: Is seed`, then the synchronization is complete. Removing synchronized files from a read-only host will *not* cause them to be resynced! Removing them from a read-write host will *cause them to be removed* from other connected peers. So, after remote users confirm that files have been transferred, either kill btsync or delete the files on the read-write server, as appropriate.

A note on key/secret distribution
=================================

With the CLI, there is currently no way to send links, or approve incoming connections. Consequently, it is best if the read only key/secret is sent in a secure manner (i.e., not through plain-text email). I should also mention that there is no way to create a single-use key or one that expires after X days. This may be possible using the API, but seemingly not currently with the `btsync` binary itself.
