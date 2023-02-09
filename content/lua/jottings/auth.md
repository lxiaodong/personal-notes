---
title: "Lua脚本实现的鉴权方式"
tags: 
- lua
- auth
- 随笔
weight: 10
description: "Lua脚本实现的鉴权方式"
---

## 1.lua脚本实现的鉴权方式
```lua
local cjson = require("cjson")

-- auth class
local Auth = {
    args = '',
    currentServer = 'mis',
    serverDeveloperMap = {},
    headers = {}
}

-- 派生类的方法
-- :是lua的语法糖，Auth:new(o) 等同于Auth.new(self, o)
function Auth:new(obj, currentServer, serverDeveloperMap)
    obj = obj or {}
    setmetatable(obj, self)
    self.__index = self
    self.currentServer = currentServer
    self.serverDeveloperMap = serverDeveloperMap
    self:init()
    return obj
end

-- 初始化
function Auth:init()
    self:initArgs()
    self:initHeaders()
end

-- 初始化请求参数
function Auth:initArgs()
    local headers = ngx.req.get_headers()
    -- 初始化args
    if "POST" == ngx.var.request_method then
        ngx.req.read_body()
        local contentType = headers["content-type"]
    	if string.sub(contentType, 1, 16) == "application/json" then
    		local dataJson = ngx.req.get_body_data()
    		self.args = dataJson
        end
    end
end

-- 初始化Headers
function Auth:initHeaders()
    self.headers = ngx.req.get_headers()
end

-- error报错
function Auth:error(code, message, data)
    -- 定义errors 格式
    local errors = {
        mis = {code = 0, msg = '', data = nil}
    }

    -- 自定义错误格式
    local errorInfoObj = {}
    for key, val in pairs(errors) do
        if key == self.currentServer then
            errorInfoObj = val
            errorInfoObj.code = code
            errorInfoObj.msg = message
            errorInfoObj.data = data
        end
    end

    local errorInfoJson = cjson.encode(errorInfoObj)
    ngx.header.content_type = "application/json"
    ngx.say(errorInfoJson)
    ngx.exit(200)
end

-- 获取auth授权参数
function Auth:getAuthParams()
    local result = nil
    for key, val in pairs(self.serverDeveloperMap) do
        if key == self.currentServer then
            if val ~= nil then
                for k, v in pairs(val) do
                    if k == self.headers.developer then
                        result = v
                        break
                    end
                end
            end
        end
    end
    return result
end

-- 验证开发者
function Auth:checkDev()
    if self:getAuthParams() == nil then
        self:error(400, 'authorization scope error')
    end
end

-- 授权验证
function Auth:checkAuth()
    local authTab = self:getAuthParams()
    local token = self.headers.token
    tmpAuthStr = 'app_id' .. authTab.app_id .. 'developer' .. self.headers.developer .. 'timestamp' .. self.headers.timestamp ..
    'body' .. self.args .. 'secret_id' .. authTab.secret_id
    if ngx.md5(tmpAuthStr) ~= token then
--         self:error(500, 'token error')
    end
end

-- token有效期验证
function Auth:checkTime()
   local time = os.time()
   if tonumber(self.headers.timestamp) < (time - 15) or tonumber(self.headers.timestamp) > (time + 15) then
--        self:error(500, 'token timeout')
   end
end

-- rewrite重写
function Auth:rewrite()
    -- tp框架跳转
    if self.currentServer == 'mis' then
        local uri = ngx.var.uri
        uri = '/index.php?s=/' .. uri
        return ngx.exec(uri)
    end
end

function Auth:check()
    -- 参数验证
    if self.headers.token == nil or self.headers.token == '' or self.headers.timestamp == nil or self.headers.timestamp == '' or self.headers.developer == '' then
        self:error(400, 'params error')
    end

    -- 验证开发者
    self:checkDev()

    -- 验证时间
    self:checkTime()

    -- 验证token
    self:checkAuth()

    -- rewrite
    self:rewrite()
end

-- 定义默认服务
local misCurrentServer = 'mis'

-- 定义serverDeveloperMap
local serverDeveloperMap = {
    mis = {
        ['athena'] = {app_id = "5b6a4c17fa1abb0d8a3808c1481d38ca", secret_id = "3d75fdec8601fcca65fc73623feee981"},
        ['acp'] = {app_id = "5b6a4c17fa1abb0d8a3808c1481d38ca", secret_id = "3d75fdec8601fcca65fc73623feee981"}
    }
}

-- 声明对象
local misAuth = Auth:new(nil, misCurrentServer, serverDeveloperMap)
misAuth:check()
```