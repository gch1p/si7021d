# si7021d

This is a Linux daemon that reads and returns data from a Si-7021 sensor. It's written
on Python, uses asyncio and was tested on Python 3.9.

It starts TCP server, accepts command `read` and returns JSON with sensor readings. 
You can send as many `read` commands during one connection as you want.

## Why?

It was created for AwesomeWM widget. The daemon is running on some Orange Pi board
with Si7021 sensor attached, the widget periodically pings it and draws temperature
and relative humidity on a wibox.

Sample widget code:
```lua
local wibox = require("wibox")
local gears = require('gears')
local awful = require("awful")
local naughty = require("naughty")
local theme = require("./themes/default/theme")
local json = require('json')
local util = require('./util')
local lgi = require('lgi')
local Gio = lgi.Gio

local SI7021_IP = '192.168.1.212'
local SI7021_PORT = 8306
local UPDATE_FREQUENCY = 2

local timer
local timer_started = false


-----------------
-- widgets stuff

local function create_image(file)
    image_widget = wibox.widget.imagebox()
    image_widget:set_resize(false)
    image_widget:set_image(file)
    return image_widget
end

local function wrap_image(image, top)
    margin = wibox.container.margin()
    margin:set_left(16)
    margin:set_top(top)
    margin:set_right(5)
    margin:set_widget(image)
    return margin
end

local function create_text(placeholder)
    text_widget = wibox.widget.textbox()
    text_widget:set_font(theme.font)
    if placeholder then
        text_widget:set_text(placeholder)
    end
    return text_widget
end


local temp_image = create_image(awful.util.getdir("config") .. "/icons/fs_01.png")
local temp_text = create_text('Connecting...')
local temp_image_wrapped = wrap_image(temp_image, 8)

local hum_image = create_image(awful.util.getdir("config") .. "/icons/humidity.png")
local hum_text = create_text()
local hum_image_wrapped = wrap_image(hum_image, 7)

local layout = wibox.layout.fixed.horizontal()
layout:add(temp_image_wrapped)
layout:add(temp_text)

local hum_visible = false
local function hum_hide()
    if not hum_visible then return end
    layout:remove_widgets(hum_image_wrapped)
    layout:remove_widgets(hum_text)
    hum_visible = false
end

local function hum_show()
    if hum_visible then return end
    layout:add(hum_image_wrapped)
    layout:add(hum_text)
    hum_visible = true
end


--------------------
--- networking stuff

local _socket
local _tcp_conn
local _istream
local _ostream

local function sensor_get_status()
    _istream = _tcp_conn:get_input_stream()
    _ostream = _tcp_conn:get_output_stream()
    
    _ostream:async_write("read\r\n")

    local buf = ''
    while true do
        local bytes = _istream:async_read_bytes(4096)

        -- it returns nil on error (such as disconnect)
        if bytes == nil then return nil end
        if not bytes then break end

        local len = bytes:get_size()
        if len > 0 then
            buf = bytes.data
            break
        end
    end

    return true, buf
end

local function sensor_update()
    Gio.Async.start(function()
        local ok, body = sensor_get_status()

        -- handle errors
        if ok == nil then
            temp_text:set_text("Connection error")
            hum_hide()
            
            _socket = nil

            timer:stop()
            timer_started = false
            
            return
        end

        local status = json.decode(body)

        hum_show()
        temp_text:set_text('' .. util.round(status.temp, 1) .. ' °C')
        hum_text:set_text('' .. util.round(status.humidity, 1) .. ' %')
    end)()
end

local function sensor_connect()
    _socket = Gio.SocketClient.new()
    _socket:set_timeout(1)

    _tcp_conn = _socket:async_connect_to_host(SI7021_IP, SI7021_PORT)

    if not _tcp_conn then
        _socket = nil
        temp_text:set_text('Connection error')
        return
    end

    timer = gears.timer({ timeout = UPDATE_FREQUENCY })
    timer:connect_signal("timeout", sensor_update)

    timer:start()
    timer_started = true

    sensor_update()
end


Gio.Async.start(sensor_connect)()

return layout
```

## License

MIT
