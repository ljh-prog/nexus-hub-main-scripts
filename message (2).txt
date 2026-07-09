--[[
    Nexus Private - Roblox Auth Wrapper (v3)
    이 스크립트를 executor에 넣으면 자동으로 인증 후 메인 스크립트를 로드합니다.
    
    사용 방법:
    1. 디스코드에서 Get Scripts 버튼으로 스크립트를 받습니다
    2. executor에 붙여넣고 실행합니다
    3. 자동으로 인증 후 메인 스크립트가 실행됩니다
    
    주의: 이 스크립트는 서버 URL과 키를 포함합니다.
    다른 사람과 공유하지 마세요!
    
    변경사항 (v3):
      - Volt executor 지원 추가
      - 더 많은 executor HTTP 함수 지원
      - HWID 감지 개선 (게임 정보 + executor 정보)
      - 디버그 출력 추가
      - script_key를 전역 변수로 변경 (Get Scripts에서 주입 가능)
]]

-- ==================== CONFIG ====================
local AUTH_SERVER = "http://58.229.197.183:8080"
script_key = script_key or "YOUR_KEY_HERE" -- Get Scripts에서 받은 키를 여기에 넣으세요

-- ==================== HWID GENERATOR ====================
local function getHWID()
    local hwidParts = {}
    
    -- executor별 HWID 함수 시도 (가장 많은 executor 지원)
    pcall(function()
        if hwid and type(hwid) == "function" then
            local h = hwid()
            if h and #h > 0 then table.insert(hwidParts, h) end
        end
    end)
    
    pcall(function()
        if gethwid and type(gethwid) == "function" then
            local h = gethwid()
            if h and #h > 0 then table.insert(hwidParts, h) end
        end
    end)
    
    -- Volt / Fluxus / 기타 executor
    pcall(function()
        if gethwid and type(gethwid) == "string" then
            table.insert(hwidParts, gethwid)
        end
    end)
    
    -- executor 식별자 추가
    pcall(function()
        if identifyexecutor then
            local name, version = identifyexecutor()
            table.insert(hwidParts, tostring(name))
        end
    end)
    
    -- Fallback: 게임 정보 기반 HWID (항상 가능)
    pcall(function()
        local players = game:GetService("Players")
        local lp = players.LocalPlayer
        if lp then
            table.insert(hwidParts, tostring(lp.UserId))
        end
    end)
    
    pcall(function()
        table.insert(hwidParts, tostring(game.GameId))
    end)
    
    pcall(function()
        table.insert(hwidParts, tostring(game.PlaceId))
    end)
    
    pcall(function()
        table.insert(hwidParts, tostring(game.JobId))
    end)
    
    -- 최종 HWID: 모든 파트를 해시
    local rawHWID = table.concat(hwidParts, "-")
    
    if #rawHWID == 0 then
        rawHWID = "fallback-" .. tostring(math.random(100000, 999999))
    end
    
    -- 해시 (지원되는 함수 우선)
    local hash = ""
    pcall(function()
        if crypt and crypt.hash then
            hash = crypt.hash(rawHWID, "sha256")
        end
    end)
    
    if hash == "" or not hash then
        pcall(function()
            if syn and syn.crypt and syn.crypt.hash then
                hash = syn.crypt.hash(rawHWID, "sha256")
            end
        end)
    end
    
    if hash == "" or not hash then
        -- Fallback: 간단한 해시
        local h = 5381
        for i = 1, #rawHWID do
            h = ((h << 5) + h + string.byte(rawHWID, i)) % 2147483647
        end
        hash = string.format("%x", h)
    end
    
    print("[Nexus] HWID parts: " .. #hwidParts .. " detected")
    print("[Nexus] Raw HWID length: " .. #rawHWID)
    print("[Nexus] Final HWID: " .. tostring(hash):sub(1, 16) .. "...")
    
    return tostring(hash)
end

-- ==================== HTTP REQUEST ====================
local function httpRequest(url, method, body)
    method = method or "GET"
    
    print("[Nexus] HTTP " .. method .. " " .. url)
    
    local success, result = pcall(function()
        -- 1. request (가장 일반적 - Synapse, Krnl, Volt 등)
        if request then
            return request({
                Url = url,
                Method = method,
                Headers = {
                    ["Content-Type"] = "application/json"
                },
                Body = body
            })
        end
        
        -- 2. http_request
        if http_request then
            return http_request({
                Url = url,
                Method = method,
                Headers = {
                    ["Content-Type"] = "application/json"
                },
                Body = body
            })
        end
        
        -- 3. syn.request (Synapse X)
        if syn and syn.request then
            return syn.request({
                Url = url,
                Method = method,
                Headers = {
                    ["Content-Type"] = "application/json"
                },
                Body = body
            })
        end
        
        -- 4. fluxus.request (Fluxus)
        if fluxus and fluxus.request then
            return fluxus.request({
                Url = url,
                Method = method,
                Headers = {
                    ["Content-Type"] = "application/json"
                },
                Body = body
            })
        end
        
        -- 5. HttpService (마지막 수단 - GET만)
        local hs = game:GetService("HttpService")
        if method == "GET" then
            -- HttpService는 POST를 지원하지 않으므로 GET만 처리
            return { Body = hs:GetAsync(url, true), StatusCode = 200 }
        end
        
        -- 6. POST를 위한 HttpService:JSONEncode + RequestAsync (일부 executor)
        if game:GetService("HttpService").RequestAsync then
            local resp = game:GetService("HttpService"):RequestAsync({
                Url = url,
                Method = method,
                Headers = { ["Content-Type"] = "application/json" },
                Body = body
            })
            return { Body = resp.Body, StatusCode = resp.StatusCode }
        end
        
        error("No HTTP method available in this executor")
    end)
    
    if success and result then
        print("[Nexus] HTTP Response: " .. tostring(result.StatusCode))
        return result.Body, result.StatusCode
    end
    
    print("[Nexus] HTTP Error: " .. tostring(result))
    return nil, 0
end

-- ==================== ERROR UI ====================
local function showErrorUI(errorMsg, isHWIDError)
    pcall(function()
        local players = game:GetService("Players")
        local lp = players.LocalPlayer
        local gui = Instance.new("ScreenGui")
        gui.Name = "NexusAuthError"
        gui.Parent = game:GetService("CoreGui")
        
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(0, 420, 0, isHWIDError and 200 or 150)
        frame.Position = UDim2.new(0.5, -210, 0.5, isHWIDError and -100 or -75)
        frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        frame.BorderSizePixel = 0
        frame.Parent = gui
        
        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 10)
        corner.Parent = frame
        
        local title = Instance.new("TextLabel")
        title.Size = UDim2.new(1, -20, 0, 30)
        title.Position = UDim2.new(0, 10, 0, 10)
        title.BackgroundTransparency = 1
        title.Text = "❌ Nexus Private - Authentication Failed"
        title.TextColor3 = Color3.fromRGB(255, 60, 60)
        title.Font = Enum.Font.GothamBold
        title.TextSize = 16
        title.TextXAlignment = Enum.TextXAlignment.Center
        title.Parent = frame
        
        local msg = Instance.new("TextLabel")
        msg.Size = UDim2.new(1, -20, 0, isHWIDError and 100 or 60)
        msg.Position = UDim2.new(0, 10, 0, 50)
        msg.BackgroundTransparency = 1
        msg.Text = errorMsg
        msg.TextColor3 = Color3.fromRGB(255, 255, 255)
        msg.Font = Enum.Font.GothamMedium
        msg.TextSize = 13
        msg.TextWrapped = true
        msg.TextXAlignment = Enum.TextXAlignment.Center
        msg.Parent = frame
        
        if isHWIDError then
            local hint = Instance.new("TextLabel")
            hint.Size = UDim2.new(1, -20, 0, 20)
            hint.Position = UDim2.new(0, 10, 0, 155)
            hint.BackgroundTransparency = 1
            hint.Text = "Go to Discord → Click HWID Reset → Try again"
            hint.TextColor3 = Color3.fromRGB(255, 200, 50)
            hint.Font = Enum.Font.GothamMedium
            hint.TextSize = 12
            hint.TextXAlignment = Enum.TextXAlignment.Center
            hint.Parent = frame
        end
        
        local close = Instance.new("TextButton")
        close.Size = UDim2.new(0, 100, 0, 30)
        close.Position = UDim2.new(0.5, -50, 1, -40)
        close.BackgroundColor3 = Color3.fromRGB(255, 60, 60)
        close.Text = "Close"
        close.TextColor3 = Color3.fromRGB(255, 255, 255)
        close.Font = Enum.Font.GothamBold
        close.TextSize = 14
        close.Parent = frame
        
        local closeCorner = Instance.new("UICorner")
        closeCorner.CornerRadius = UDim.new(0, 6)
        closeCorner.Parent = close
        
        close.MouseButton1Click:Connect(function()
            gui:Destroy()
        end)
        
        task.delay(20, function()
            if gui.Parent then gui:Destroy() end
        end)
    end)
end

-- ==================== AUTHENTICATION ====================
local function authenticate()
    local hwid = getHWID()
    
    print("[Nexus] Authenticating...")
    print("[Nexus] Key: " .. script_key)
    
    local body = game:GetService("HttpService"):JSONEncode({
        key = script_key,
        hwid = hwid
    })
    
    local responseBody, statusCode = httpRequest(AUTH_SERVER .. "/api/auth", "POST", body)
    
    if not responseBody or statusCode ~= 200 then
        -- 에러 메시지 파싱
        local errorMsg = "Connection failed (status: " .. tostring(statusCode) .. ")"
        local errorCode = nil
        if responseBody then
            pcall(function()
                local parsed = game:GetService("HttpService"):JSONDecode(responseBody)
                if parsed.error then errorMsg = parsed.error end
                if parsed.code then errorCode = parsed.code end
            end)
        end
        
        print("[Nexus] ❌ Authentication failed: " .. errorMsg)
        return false, errorMsg, errorCode
    end
    
    -- 응답 파싱
    local success, parsed = pcall(function()
        return game:GetService("HttpService"):JSONDecode(responseBody)
    end)
    
    if not success or not parsed.success then
        local errorMsg = (parsed and parsed.error) or "Unknown error"
        local errorCode = (parsed and parsed.code) or nil
        print("[Nexus] ❌ Authentication failed: " .. errorMsg)
        return false, errorMsg, errorCode
    end
    
    print("[Nexus] ✅ Authentication successful!")
    print("[Nexus] Session: " .. (parsed.session_token or "N/A"))
    
    return true, parsed
end

-- ==================== LOAD SCRIPT ====================
local function loadMainScript(authData)
    if not authData or not authData.script_url then
        print("[Nexus] ❌ No script URL received")
        return false
    end
    
    print("[Nexus] 📜 Loading script from: " .. authData.script_url)
    
    local scriptContent, statusCode = httpRequest(authData.script_url, "GET")
    
    if not scriptContent or statusCode ~= 200 then
        print("[Nexus] ❌ Failed to load script (status: " .. tostring(statusCode) .. ")")
        return false
    end
    
    -- 스크립트 실행
    local success, err = pcall(function()
        local func = loadstring(scriptContent)
        if func then
            func()
        else
            error("Failed to compile script")
        end
    end)
    
    if not success then
        print("[Nexus] ❌ Script execution error: " .. tostring(err))
        return false
    end
    
    print("[Nexus] ✅ Script loaded successfully!")
    return true
end

-- ==================== HEARTBEAT LOOP ====================
local sessionToken = nil
local function startHeartbeat()
    if not sessionToken then return end
    
    task.spawn(function()
        while true do
            task.wait(300) -- 5분마다 하트비트
            local body = game:GetService("HttpService"):JSONEncode({
                session_token = sessionToken
            })
            httpRequest(AUTH_SERVER .. "/api/heartbeat", "POST", body)
        end
    end)
end

-- ==================== MAIN ====================
local function main()
    print("[Nexus] ===================================")
    print("[Nexus] Nexus Private Auth Wrapper v3")
    print("[Nexus] ===================================")
    
    -- 인증
    local success, result, errorCode = authenticate()
    
    if not success then
        -- HWID 불일치인지 확인
        local isHWIDError = (errorCode == "HWID_MISMATCH")
        
        if isHWIDError then
            showErrorUI(
                "⚠️ HWID Mismatch!\n\nThis key is bound to another device.\nTo use this script on this device, go to the Discord server and click the HWID Reset button.",
                true
            )
        else
            showErrorUI(result or "Unknown error", false)
        end
        return
    end
    
    -- 세션 토큰 저장
    sessionToken = result.session_token
    startHeartbeat()
    
    -- 메인 스크립트 로드
    loadMainScript(result)
end

-- 실행
main()
