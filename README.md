# nginx-rtmp-live-stream

## Download, build and install

Build Utilities

    sudo apt-get install build-essential libpcre3 libpcre3-dev libssl-dev git zlib1g-dev -y

## Set up live streaming

To set up RTMP support you need to add `rtmp{}` section to `nginx.conf` (can be found in PREFIX/conf/nginx.conf). Stock `nginx.conf` contains only `http{}` section.

    sudo atom /usr/local/nginx/conf/nginx.conf

Use this `nginx.conf` instead of stock config:
          
          user  nginx;
          worker_processes  1;

          error_log  /var/log/nginx/error.log warn;
          pid        /var/run/nginx.pid;

          load_module modules/ngx_rtmp_module.so;
          load_module modules/ngx_http_vod_module.so;

          events {
              worker_connections  1024;
          }

          rtmp_auto_push on;
          rtmp {
              server {
                  listen 1935;
                  chunk_size 4000;

                  play_time_fix off;
                  interleave on;
                  publish_time_fix on;

                  application live {
                      live on;
                      # Turn on HLS
                      hls on;
                      hls_path /tmp/hls;
                      hls_fragment 3;
                      hls_playlist_length 60;
                  }
              }
          }

          http {
              sendfile off;
              tcp_nopush on;
              aio on;
              directio 512;
              default_type application/octet-stream;

              server {
                  listen 3000;

                  location /hls {
                      # Disable cache
                      add_header 'Cache-Control' 'no-cache';

                      # CORS setup
                      add_header 'Access-Control-Allow-Origin' '*' always;
                      add_header 'Access-Control-Expose-Headers' 'Content-Length';

                      # allow CORS preflight requests
                      if ($request_method = 'OPTIONS') {
                          add_header 'Access-Control-Allow-Origin' '*';
                          add_header 'Access-Control-Max-Age' 1728000;
                          add_header 'Content-Type' 'text/plain charset=UTF-8';
                          add_header 'Content-Length' 0;
                          return 204;
                      }

                      types {
                          application/dash+xml mpd;
                          application/vnd.apple.mpegurl m3u8;
                          video/mp2t ts;
                      }

                      root /tmp/;
                  }

                  # vod caches
                  vod_metadata_cache metadata_cache 256m;
                  vod_response_cache response_cache 128m;

                  # vod settings
                  vod_mode local;
                  vod_segment_duration 2000; # 2s
                  vod_align_segments_to_key_frames on;

                  #file handle caching / aio
                  open_file_cache max=1000 inactive=5m;
                  open_file_cache_valid 2m;
                  open_file_cache_min_uses 1;
                  open_file_cache_errors on;
                  aio on;

                  location /video/ {
                      alias /home/nginx-local-videos/;
                      vod hls;
                      add_header Access-Control-Allow-Headers '*';
                      add_header Access-Control-Expose-Headers 'Server,range,Content-Length,Content-Range';
                      add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
                      add_header Access-Control-Allow-Origin '*';
                      expires 100d;
                  }
              }
          }


Restart nginx with:

    sudo /usr/local/nginx/sbin/nginx -s stop
    sudo /usr/local/nginx/sbin/nginx
