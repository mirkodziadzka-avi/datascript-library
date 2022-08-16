# Custom Site Persistence using Cookie - Solution A

Site Persistence can be achieved leveraging built-in Avi GSLB functionality, however there are use cases when external GSLB is used and site persistence required with Avi solution.

Solution A:
1. Cookie "site-cookie" is leveraged to define site persistence by allocating the cookie on new request that identify site specific information, site_cookie is defined as site_cookie = avi.vs.name() .. ":" .. avi.vs.ip() .. ":" .. avi.vs.port()
2. There are two pools has to be defined per site local_pool and proxy_pool:
   a. local_pool will be used to serve new requests locally or redirected requests from other sites based on decisions made at remote sites. local_pool includes actual servers.
   b. proxy_pool will be used to redirect requests to site defined in cookie or will be pool to handle request if there are no local pool members considered up. proxy_pool includes other sites Virtual Services IPs.
3. There are list of health checks built-in:
   a. to avoid redirecting request to site with no active servers in local_pool
   b. to avoid attaching cookie for local processing if there are no local_pool members up
   c. serve a maintenance page when both local and proxy pool don't have any members up.

```lua
-- HTTP_REQ
avi.vs.reqvar.cookie_private_key= "7762135A21351139927D442559CE06CC"
proxy_pool = "proxy_pool"
local_pool = "local_pool"

-- Globals
local_pool_servers_up_count, local_pool_servers_total_count = avi.pool.get_servers(local_pool)
proxy_pool_servers_up_count, proxy_pool_servers_total_count = avi.pool.get_servers(proxy_pool)
cookie_exists = nil
decrypted_site_cookie = nil
avi.vs.reqvar.maintenance_mode = nil
avi.vs.reqvar.set_cookie = nil

if avi.http.cookie_exists("site-cookie") then
  cookie_exists = 1
  site_cookie = avi.http.get_cookie("site-cookie")
  decoded_site_cookie = avi.utils.base64_decode(site_cookie)
  valid_block_length = pcall(avi.crypto.decrypt,decoded_site_cookie,avi.vs.reqvar.cookie_private_key)
  if valid_block_length then
    decrypted_site_cookie = avi.crypto.decrypt(decoded_site_cookie,avi.vs.reqvar.cookie_private_key)
  else
    avi.http.remove_cookie("site-cookie")
    decrypted_site_cookie = nil
  end
end

if cookie_exists and decrypted_site_cookie then
  site_cookie = {}
  for match in decrypted_site_cookie:gmatch("([^:]+),?") do
          table.insert(site_cookie,match)
  end
  -- decrypted_site_cookie = "VS1:192.168.0.246:80"
  -- site_cookie[1] = VS1
  -- site_cookie[2] = 192.168.0.246
  -- site_cookie[3] = 80

  cookie_site_id = site_cookie[1]
  cookie_proxy_pool_server_address = site_cookie[2]
  cookie_proxy_pool_server_port = site_cookie[3]


  if cookie_site_id == avi.vs.name() then
    -- if local request, check local and proxy pools are operational, if not force maintenance
    if local_pool_servers_up_count < 1 and proxy_pool_servers_up_count < 1 then
      avi.vs.reqvar.maintenance_mode = 1
    -- if local request, the request has to be served by local pool, however if local pool does not have any active members, request has to be send to proxy pool
    elseif local_pool_servers_up_count < 1 then
      avi.http.remove_cookie("site-cookie")
      avi.pool.select(proxy_pool)
    -- if local request and local pool has operational members, serve the request
    else
      avi.vs.reqvar.set_cookie = 1
      avi.pool.select(local_pool)
    end
  else
    -- if request is not designed for local site, check local and proxy pools are operational, if not force maintenance
    if local_pool_servers_up_count < 1 and proxy_pool_servers_up_count < 1 then
      avi.vs.reqvar.maintenance_mode = 1
      -- if request is not designed for local site, the request has to be served by proxy pool, however if proxy pool does not have any active members, request has to be send to local pool
    elseif proxy_pool_servers_up_count < 1 then
    -- set a new persistence cookie on HTTP_RESP
    avi.vs.reqvar.set_cookie = 1
    avi.http.remove_cookie("site-cookie")
    avi.pool.select(local_pool)
    -- if request is not designed for local site and proxy pool has operational members, redirect the request to original server
    else
      -- validate if provided proxy pool server is part of proxy pool
      cookie_proxy_pool_server_exists = pcall(avi.pool.select,proxy_pool,cookie_proxy_pool_server_address)
      if cookie_proxy_pool_server_exists then
        avi.pool.select(proxy_pool, cookie_proxy_pool_server_address)
      else
        avi.vs.reqvar.set_cookie = 1
        avi.http.remove_cookie("site-cookie")
        avi.pool.select(local_pool)
      end
    end
  end
else
  -- if no cookie, check if proxy or local pools have active members, if not force maintenance
  if local_pool_servers_up_count < 1 and proxy_pool_servers_up_count < 1 then
    avi.vs.reqvar.maintenance_mode = 1
  -- if no cookie, and local pool does not have active members, redirect request to proxy pool
  elseif local_pool_servers_up_count < 1 then
    avi.pool.select(proxy_pool)
  else
  -- if no cookie, serve request locally
  -- set a new persistence cookie on HTTP_RESP
    avi.vs.reqvar.set_cookie = 1
    avi.pool.select(local_pool)
  end
end
```

```lua
-- HTTP_RESP
if avi.vs.reqvar.maintenance_mode == 1 then
  avi.http.response(503, {content_type="text/html"},"Maintenance")
else
  if avi.vs.reqvar.set_cookie == 1 then
    site_cookie = avi.vs.name() .. ":" .. avi.vs.ip() .. ":" .. avi.vs.port()
    encrypted_site_cookie = avi.crypto.encrypt(site_cookie, avi.vs.reqvar.cookie_private_key)
    encoded_site_cookie = avi.utils.base64_encode(encrypted_site_cookie)
    avi.http.add_cookie ("site-cookie", encoded_site_cookie)
  end
end
```
