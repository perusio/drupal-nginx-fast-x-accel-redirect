# Nginx Fast Private File Transfer for Drupal using X-Accel-Redirect 

## Installation (for the impatient)
 
 1. Install the module as usual.
 
 2. Go to `admin/settings/file-system` and enable **private files**. 
 
 3. Go to `admin/settings/file-system/nginx-accel-redirect` and enable
    the **Files transferred by Nginx using X-Accel-Redirect**.
    
 4. Copy the suggested configuration at
    `admin/settings/file-system/nginx-accel-redirect` from the
    textarea to your vhost (server block) configuration.
    
 5. Done. 
 
 6. Read the discussion below to troubleshoot or gain a deeper
    understanding of what the module does and what's at stake.
 
## Background

[Lighty X-Sendfile](http://blog.lighttpd.net/articles/2006/07/02/x-sendfile
"Lighty's life blog post on X-Sendfile")
was the first implementation of a mechanism for serving private files
fast. By **private** I mean files that require some type of
authentication with the backend. 

The mechanism consists in a **header** &mdash; `X-Sendfile` in
lighty's case &mdash; that is sent from the backend
and that includes the _real_ file path. When receiving this header the
server discards the backend reply and serves the file **directly**.

This is obviously a _dangerous_ thing to do. Since if not properly
setup all the private file system becomes exposed to the web.

Hence it must be enabled on a case by case basis.

## Nginx X-Sendfile Implementation

In [Nginx](http://wiki.nginx.org "Nginx Wiki") the `X-Sendfile` header is called
[X-Accel-Redirect](http://wiki.nginx.org/XSendfile "Nginx
implementation of X-Sendfile"). Contrary to what
happens in Apache that requires the
[X-Sendfile](https://tn123.org/mod_xsendfile/ "Apache X-Sendfile") to module be installed
and enabled in the server config, Nginx provides this facility in
**core**, i.e., _out of the box_.


## Nginx X-Accel-Redirect and Drupal

To take advantage of this module you must:

 1. Be running Nginx either as the main server or as reverse proxy
    (handling all static files).
    
 2. Have **private** files enabled in your filesystem settings page,
    located at `admin/settings/file-system`.
    
In the filesystem settings page you specify the path to the private
files directory relative to the Drupal web root, i.e., the base
directory where Drupal is installed.

Setting to `private` means that the files will be stored in the
`private` directory of your Drupal install. If installed under
`/var/www/drupal-site/` the private files will be under
`/var/www/drupal-site/protected`. Hence this directory must be
**writable** by the web server.

You must **protect** the private files directory of web access,
meaning this directory shouldn't be accessible over the web. For that
you must set this **location** in your Nginx configuration as
[internal](http://wiki.nginx.org/NginxHttpCoreModule#internal). This
guarantees that the files are accessible only by the server. No
external access is possible.

In the settings form at
`admin/settings/file-system/nginx-accel-redirect` there's a suggested
configuration for protecting the private files directory. Assuming
that you're having the private files directory at `/files/private` the
suggested config will be:

    location ~* /files/private/ {
      internal;
    }
    
Also you must relay the `system/files/` path to Drupal in your Nginx
config. The module also suggests a configuration stanza for that at
the settings form at
`admin/settings/file-system/nginx-accel-redirect`.

The exact configuration details will vary with your setup but
if you use a configuration like
[mine](https://github.com/perusio/drupal-with-nginx "My Nginx config
on github") this will be already contemplated. Most configs that are
referenced in the groups.drupal.org
[Nginx](http://groups.drupal.org/nginx) group accommodate private
files handling with a location stanza like this:

    location ~* /system/files/ {
      try_files $uri /index.php?q=$uri&$args;
      log_not_found off;
    }

if you're on **drupal 7**, or on drupal 6 not using anything that uses
[custom\_url\_rewrite\_outbound](http://api.drupal.org/api/drupal/developer--hooks--core.php/function/custom_url_rewrite_outbound/6).  This excludes sites using the [purl](http://drupal.org/project/purl)
module or other module relying on it like
[spaces](http://drupal.org/project/spaces). Two very popular
drupal based projects make use of it:
[OpenAtrium](http://openatrium.com) and
[ManagingNews](http://managingnews.com). In that case use:

    location ~* /system/files/ {
       try_files $uri @drupal;
    }

where `@drupal` is the
[`named location`](https://github.com/perusio/drupal-with-nginx/blob/master/sites-available/drupal_spaces.conf)
that invokes Drupal `index.php` with the proper query string and arguments.

Note that by default nginx **logs the _missing_ files** as a 404
error. That's how it's supposed to be. The file doesn't exist and in
that case relay the request to the named location `@drupal`. If 
you don't want that to happen then add `log_not_found off` to the
above location stanza. Like this:

    location ~* /system/files/ {
       try_files $uri /index.php?q=$uri&$args;
       log_not_found off;
    }
or
  
    location ~* /system/files/ {
       try_files $uri @drupal
       log_not_found off;
    }    
depending on the above discussed situation.

## Security considerations

The module generates a status line at the status page
`/admin/reports/status` stating if the private files directory is
accessible from the web. 

If you're paranoid then you can **double check** with the following
commands, using `curl`:

    curl -I <URI to private file>
    
or alternatively with `wget`
     
    wget -S <URI to private file>
 
If you have a private file named `foobar.pdf` in your private file
directory located at `private`, when issuing:
 
    curl -I http://example.com/private/foobar.pdf

you should get a `404 File Not Found` status
code. `http://example.com` being the base url of your Drupal site.
If not then check your Nginx config for blocking direct access to the
private files as described above.
 
 
## Testing of directory protection

The testing of the protection is done by touching a file in the
private file directory with a name like
`00nginx_accel_redirect4e04b170247834.21219541` where the trailing
alphanumeric substring is generated by
[`uniqid`](http://php.net/manual/en/function.uniqid.php). 
 
## Acknowledgments

The author of the [xsend](http://drupal.org/project/xsend "xsend
Drupal module") module. There's a lot of stuff stolen from there in
this module.

The gang on the [Nginx](http://groups.drupal.org/nginx) Drupal group,
in particular this [discussion](http://groups.drupal.org/node/36892)
on how to adapt the `xsend` module, which is for Apache, to be used by
Nginx.
