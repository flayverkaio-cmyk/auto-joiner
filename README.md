local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local Player = Players.LocalPlayer

local THEME = {
    Background    = Color3.fromRGB(10, 10, 10),
    SidebarBG     = Color3.fromRGB(18, 18, 18),
    ItemBG        = Color3.fromRGB(30, 30, 30),
    Stroke        = Color3.fromRGB(74, 74, 74),
    Accent        = Color3.fromRGB(255, 204, 48),
    DiscordAccent = Color3.fromRGB(255, 204, 48),
    TextWhite     = Color3.fromRGB(246, 246, 246),
    TextGray      = Color3.fromRGB(170, 170, 170),
    GreenActive   = Color3.fromRGB(52, 211, 153),
    RedSoft       = Color3.fromRGB(255, 120, 120),
}

local FIREBASE_URL  = "https://notify-f6091-default-rtdb.firebaseio.com/.json"
local POLL_INTERVAL = 1.0
local MIN_VAL       = 1e6

local State = {
    AutoJoin     = false,
    MinGenFilter = MIN_VAL,
    ProcessedIds = {},
}

-- =============================================================================
-- HTTP ROBUSTO (Synapse, KRNL, Fluxus, Delta, Codex, etc.)
-- =============================================================================
local function httpRequest(url)
    local body = nil
    local reqFunc = (syn and syn.request) or http_request or request

    if reqFunc then
        local ok, res = pcall(reqFunc, {
            Url = url,
            Method = "GET",
            Headers = {
                ["User-Agent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
                ["Accept"] = "application/json"
            }
        })
        if ok then
            if type(res) == "table" and res.Body then
                body = res.Body
            elseif type(res) == "string" then
                body = res
            end
        end
    end

    if not body then
        local ok, res = pcall(function()
            return game:HttpGet(url, true)
        end)
        if ok and res then body = res end
    end

    return body
end

-- =============================================================================
-- UTILIDADES
-- =============================================================================
local function formatNumber(n)
    n = tonumber(n) or 0
    if n >= 1e12 then return string.format("%.2fT", n / 1e12)
    elseif n >= 1e9  then return string.format("%.2fB", n / 1e9)
    elseif n >= 1e6  then return string.format("%.2fM", n / 1e6)
    elseif n >= 1e3  then return string.format("%.2fK", n / 1e3) end
    return tostring(math.floor(n))
end

local function parseMoneyValue(value)
    if value == nil then return 0 end
    if type(value) == "number" then return value end

    local s = tostring(value)
    s = s:gsub("[%s$]", "")

    if s:match("^%d+,%d+[KkMmBbTt]$") then
        s = s:gsub(",", ".")
    else
        s = s:gsub(",", "")
    end

    local numStr = s:match("[%d%.]+")
    local num = tonumber(numStr)
    if not num then return 0 end

    local u = s:upper()
    if u:find("T") then return num * 1e12
    elseif u:find("B") then return num * 1e9
    elseif u:find("M") then return num * 1e6
    elseif u:find("K") then return num * 1e3 end
    return num
end

local function getField(tbl, keys)
    if type(tbl) ~= "table" then return nil end
    for _, k in ipairs(keys) do
        local v = tbl[k]
        if v ~= nil and v ~= "" then return v end
    end
    return nil
end

-- =============================================================================
-- NORMALIZAÇÃO DE DADOS DO FIREBASE
-- =============================================================================
local function normalizeEntry(fbKey, rawData)
    if type(rawData) ~= "table" then return nil end

    local name     = getField(rawData, {"name","Name","nome","Nome","player","Player","username","Username","user","User","displayName","DisplayName"}) or "Unknown"
    local moneyRaw = getField(rawData, {"money","Money","value","Value","gen","Gen","generations","Generations","amount","Amount","cash","Cash","total","Total"})
    local jobId    = getField(rawData, {"jobId","JobId","jobID","JobID","serverId","serverID","job_id","id","Id","ID","server","Server","joinId","JoinId","job","Job","server_id"})
    local placeId  = tonumber(getField(rawData, {"placeId","PlaceId","placeID","PlaceID","place_id","gameId","GameId","game_id"})) or game.PlaceId

    if not jobId and rawData.join_script then
        jobId = tostring(rawData.join_script):match('["\']?([%w%-]+)["\']?')
    end
    if not jobId and rawData.link then
        jobId = tostring(rawData.link):match('JobId=([%w%-]+)') or tostring(rawData.link):match('([%w%-]+)$')
    end
    if not jobId and rawData.url then
        jobId = tostring(rawData.url):match('([%w%-]+)$')
    end

    if not jobId then
        return nil
    end

    local numVal   = parseMoneyValue(moneyRaw)
    local uniqueId = tostring(fbKey) .. "_" .. tostring(jobId)

    return {
        fbKey      = tostring(fbKey),
        uniqueId   = uniqueId,
        nome       = tostring(name),
        money      = (type(moneyRaw) == "string" and moneyRaw ~= "") and moneyRaw or ("$" .. formatNumber(numVal)),
        valor      = numVal,
        jobId      = tostring(jobId),
        placeId    = placeId,
    }
end

local function collectAll(snapshot)
    local out = {}
    if type(snapshot) ~= "table" then return out end

    for fbKey, rawData in pairs(snapshot) do
        if type(rawData) == "table" then
            local item = normalizeEntry(fbKey, rawData)
            if item then
                out[item.uniqueId] = item
            else
                for subKey, subData in pairs(rawData) do
                    if type(subData) == "table" then
                        local subItem = normalizeEntry(subKey, subData)
                        if subItem then
                            out[subItem.uniqueId] = subItem
                        end
                    end
                end
            end
        end
    end
    return out
end

-- =============================================================================
-- CONSTRUÇÃO DA INTERFACE
-- =============================================================================
for _, c in pairs(CoreGui:GetChildren()) do
    if c.Name == "CanelloniJoinerUI" then c:Destroy() end
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CanelloniJoinerUI"
ScreenGui.ResetOnSpawn = false
local ok = pcall(function() ScreenGui.Parent = CoreGui end)
if not ok then ScreenGui.Parent = Player:WaitForChild("PlayerGui") end

local OPEN_POS = UDim2.new(0.5, -225, 0.5, -175)

local BorderShell = Instance.new("Frame")
BorderShell.Name = "BorderShell"
BorderShell.Parent = ScreenGui
BorderShell.Size = UDim2.new(0, 458, 0, 428)
BorderShell.Position = UDim2.new(OPEN_POS.X.Scale, OPEN_POS.X.Offset - 4, OPEN_POS.Y.Scale, OPEN_POS.Y.Offset - 4)
BorderShell.BackgroundTransparency = 1
Instance.new("UICorner", BorderShell).CornerRadius = UDim.new(0, 14)

local ShellStroke = Instance.new("UIStroke", BorderShell)
ShellStroke.Thickness = 2
ShellStroke.Transparency = 0.08
ShellStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

local BorderGradient = Instance.new("UIGradient", ShellStroke)
BorderGradient.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0,   THEME.Stroke),
    ColorSequenceKeypoint.new(0.5, THEME.Accent),
    ColorSequenceKeypoint.new(1,   THEME.Stroke),
}
task.spawn(function()
    local angle = 0
    while BorderShell and BorderShell.Parent do
        angle = (angle + RunService.Heartbeat:Wait() * 60) % 360
        pcall(function() BorderGradient.Rotation = angle end)
    end
end)

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.Size = UDim2.new(0, 450, 0, 420)
MainFrame.Position = OPEN_POS
MainFrame.BackgroundColor3 = THEME.Background
MainFrame.ClipsDescendants = true
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 12)

local MainStroke = Instance.new("UIStroke", MainFrame)
MainStroke.Color = THEME.Stroke
MainStroke.Thickness = 1
MainStroke.Transparency = 0.7

local Header = Instance.new("Frame")
Header.Name = "Header"
Header.Parent = MainFrame
Header.Size = UDim2.new(1, 0, 0, 105)
Header.BackgroundTransparency = 1

local DragHandle = Instance.new("Frame")
DragHandle.Name = "DragHandle"
DragHandle.Parent = Header
DragHandle.Size = UDim2.new(1, 0, 0, 34)
DragHandle.BackgroundTransparency = 1
DragHandle.Active = true

local TitleLabel = Instance.new("TextLabel")
TitleLabel.Parent = Header
TitleLabel.Text = "Canelloni Joiner"
TitleLabel.Font = Enum.Font.GothamBlack
TitleLabel.TextSize = 16
TitleLabel.TextColor3 = THEME.TextWhite
TitleLabel.Size = UDim2.new(0, 220, 0, 25)
TitleLabel.Position = UDim2.new(0, 20, 0, 10)
TitleLabel.BackgroundTransparency = 1
TitleLabel.TextXAlignment = Enum.TextXAlignment.Left

local Controls = Instance.new("Frame")
Controls.Name = "Controls"
Controls.Parent = Header
Controls.Size = UDim2.new(1, -40, 0, 32)
Controls.Position = UDim2.new(0, 20, 0, 52)
Controls.BackgroundTransparency = 1

local AJButton = Instance.new("TextButton")
AJButton.Name = "AJButton"
AJButton.Parent = Controls
AJButton.Text = "AUTO JOIN: OFF"
AJButton.Size = UDim2.new(0, 130, 1, 0)
AJButton.BackgroundColor3 = THEME.ItemBG
AJButton.TextColor3 = THEME.TextWhite
AJButton.Font = Enum.Font.GothamBold
AJButton.TextSize = 10
Instance.new("UICorner", AJButton).CornerRadius = UDim.new(0, 8)

local ValueInput = Instance.new("TextBox")
ValueInput.Name = "ValueInput"
ValueInput.Parent = Controls
ValueInput.Text = "1M"
ValueInput.Size = UDim2.new(1, -140, 1, 0)
ValueInput.Position = UDim2.new(0, 140, 0, 0)
ValueInput.BackgroundColor3 = THEME.ItemBG
ValueInput.TextColor3 = THEME.TextWhite
ValueInput.Font = Enum.Font.GothamBold
ValueInput.TextSize = 11
ValueInput.ClearTextOnFocus = false
Instance.new("UICorner", ValueInput).CornerRadius = UDim.new(0, 8)

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Name = "StatusLabel"
StatusLabel.Parent = Header
StatusLabel.Size = UDim2.new(1, -40, 0, 20)
StatusLabel.Position = UDim2.new(0, 20, 0, 86)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Connecting..."
StatusLabel.TextColor3 = THEME.Accent
StatusLabel.TextSize = 11
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left

local Scroll = Instance.new("ScrollingFrame")
Scroll.Name = "Scroll"
Scroll.Parent = MainFrame
Scroll.Size = UDim2.new(1, -40, 1, -175)
Scroll.Position = UDim2.new(0, 20, 0, 115)
Scroll.BackgroundTransparency = 1
Scroll.ScrollBarThickness = 4
Scroll.ScrollBarImageColor3 = THEME.Accent
Scroll.ClipsDescendants = true

local ListLayout = Instance.new("UIListLayout")
ListLayout.Parent = Scroll
ListLayout.Padding = UDim.new(0, 8)
ListLayout.SortOrder = Enum.SortOrder.LayoutOrder
ListLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    Scroll.CanvasSize = UDim2.new(0, 0, 0, ListLayout.AbsoluteContentSize.Y + 10)
end)

local DiscordButton = Instance.new("Frame")
DiscordButton.Name = "DiscordButton"
DiscordButton.Parent = ScreenGui
DiscordButton.Size = UDim2.new(0, 450, 0, 32)
DiscordButton.Position = UDim2.new(OPEN_POS.X.Scale, OPEN_POS.X.Offset, OPEN_POS.Y.Scale, OPEN_POS.Y.Offset + 428)
DiscordButton.BackgroundColor3 = THEME.SidebarBG
Instance.new("UICorner", DiscordButton).CornerRadius = UDim.new(0, 8)

local DiscordLabel = Instance.new("TextLabel")
DiscordLabel.Parent = DiscordButton
DiscordLabel.Size = UDim2.new(1, 0, 1, 0)
DiscordLabel.BackgroundTransparency = 1
DiscordLabel.Text = "Discord: discord.gg/chJNxmKgVY"
DiscordLabel.TextColor3 = THEME.DiscordAccent
DiscordLabel.TextSize = 11
DiscordLabel.Font = Enum.Font.GothamBold

local DiscordClick = Instance.new("TextButton")
DiscordClick.Parent = DiscordButton
DiscordClick.Size = UDim2.new(1, 0, 1, 0)
DiscordClick.BackgroundTransparency = 1
DiscordClick.Text = ""
DiscordClick.MouseButton1Click:Connect(function()
    pcall(function() setclipboard("discord.gg/chJNxmKgVY") end)
end)

local ToggleFrame = Instance.new("Frame")
ToggleFrame.Name = "ToggleFrame"
ToggleFrame.Parent = ScreenGui
ToggleFrame.Size = UDim2.new(0, 45, 0, 45)
ToggleFrame.Position = UDim2.new(0, 10, 0.5, -22)
ToggleFrame.BackgroundColor3 = THEME.Background
Instance.new("UICorner", ToggleFrame).CornerRadius = UDim.new(0, 10)

local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Parent = ToggleFrame
ToggleBtn.Text = "CJ"
ToggleBtn.Size = UDim2.new(1, 0, 1, 0)
ToggleBtn.BackgroundTransparency = 1
ToggleBtn.TextColor3 = THEME.Accent
ToggleBtn.Font = Enum.Font.GothamBlack
ToggleBtn.TextSize = 14
ToggleBtn.MouseButton1Click:Connect(function()
    local vis = not MainFrame.Visible
    MainFrame.Visible = vis
    BorderShell.Visible = vis
    DiscordButton.Visible = vis
end)

-- =============================================================================
-- LÓGICA DA UI
-- =============================================================================
local function setStatus(txt, color)
    pcall(function()
        StatusLabel.Text = txt
        StatusLabel.TextColor3 = color or THEME.TextGray
    end)
end

local function createRow(data)
    local Row = Instance.new("Frame")
    Row.Name = "Row_" .. data.uniqueId
    Row.Parent = Scroll
    Row.Size = UDim2.new(1, 0, 0, 58)
    Row.BackgroundColor3 = THEME.ItemBG
    Instance.new("UICorner", Row).CornerRadius = UDim.new(0, 8)
    Instance.new("UIStroke", Row).Color = THEME.Stroke

    local NameL = Instance.new("TextLabel")
    NameL.Parent = Row
    NameL.Text = data.nome
    NameL.Size = UDim2.new(0, 170, 0, 22)
    NameL.Position = UDim2.new(0, 12, 0, 7)
    NameL.Font = Enum.Font.GothamBold
    NameL.TextSize = 12
    NameL.TextColor3 = THEME.TextWhite
    NameL.BackgroundTransparency = 1
    NameL.TextXAlignment = Enum.TextXAlignment.Left

    local ValueL = Instance.new("TextLabel")
    ValueL.Parent = Row
    ValueL.Text = data.money
    ValueL.Size = UDim2.new(0, 95, 0, 22)
    ValueL.Position = UDim2.new(0, 190, 0, 7)
    ValueL.Font = Enum.Font.GothamBold
    ValueL.TextColor3 = THEME.Accent
    ValueL.TextSize = 13
    ValueL.BackgroundTransparency = 1
    ValueL.TextXAlignment = Enum.TextXAlignment.Left

    local jobShort = #data.jobId > 20 and (data.jobId:sub(1, 20) .. "...") or data.jobId
    local JobLabel = Instance.new("TextLabel")
    JobLabel.Parent = Row
    JobLabel.Text = "JobId: " .. jobShort
    JobLabel.Size = UDim2.new(1, -90, 0, 18)
    JobLabel.Position = UDim2.new(0, 12, 0, 31)
    JobLabel.Font = Enum.Font.Gotham
    JobLabel.TextSize = 10
    JobLabel.TextColor3 = THEME.TextGray
    JobLabel.BackgroundTransparency = 1
    JobLabel.TextXAlignment = Enum.TextXAlignment.Left

    local Join = Instance.new("TextButton")
    Join.Parent = Row
    Join.Text = "JOIN"
    Join.Size = UDim2.new(0, 58, 0, 30)
    Join.Position = UDim2.new(1, -68, 0.5, -15)
    Join.BackgroundColor3 = THEME.Accent
    Join.TextColor3 = THEME.TextWhite
    Join.Font = Enum.Font.GothamBold
    Join.TextSize = 11
    Instance.new("UICorner", Join).CornerRadius = UDim.new(0, 6)
    Join.MouseButton1Click:Connect(function()
        setStatus("Joining: " .. data.nome, THEME.Accent)
        local ok2, err2 = pcall(function()
            TeleportService:TeleportToPlaceInstance(data.placeId or game.PlaceId, data.jobId, Player)
        end)
        if not ok2 then
            setStatus("Teleport failed!", THEME.RedSoft)
        end
    end)
end

-- =============================================================================
-- FETCH E LOOP
-- =============================================================================
local function fetch()
    local cacheBuster = tostring(math.random(100000, 999999)) .. "_" .. tostring(tick())
    local url = FIREBASE_URL .. "?t=" .. cacheBuster
    local body = httpRequest(url)

    if not body or body == "" then
        return nil
    end

    if body:lower() == "null" then
        return nil
    end

    local ok, decoded = pcall(HttpService.JSONDecode, HttpService, body)
    if ok and type(decoded) == "table" then
        return decoded
    else
        return nil
    end
end

State.MinGenFilter = parseMoneyValue(ValueInput.Text)

AJButton.MouseButton1Click:Connect(function()
    State.AutoJoin = not State.AutoJoin
    AJButton.Text = "AUTO JOIN: " .. (State.AutoJoin and "ON" or "OFF")
    AJButton.BackgroundColor3 = State.AutoJoin and THEME.Accent or THEME.ItemBG
end)

ValueInput.FocusLost:Connect(function()
    local v = parseMoneyValue(ValueInput.Text)
    if v >= 0 then
        State.MinGenFilter = v
        ValueInput.Text = (v == 0) and "0" or formatNumber(v)
        setStatus("Filter: " .. ValueInput.Text, THEME.Accent)
    else
        ValueInput.Text = formatNumber(State.MinGenFilter)
    end
end)

task.spawn(function()
    local failCount = 0
    local itemCount = 0
    local connected = false
    local IsFirstFetch = true -- << NOVO: ignora histórico no primeiro ciclo

    while true do
        task.wait(POLL_INTERVAL)

        local currentData = fetch()

        if currentData then
            if not connected then
                connected = true
                setStatus("Connected", THEME.GreenActive)
            end

            failCount = 0
            local current = collectAll(currentData)

            for uniqueId, item in pairs(current) do
                if not State.ProcessedIds[uniqueId] then
                    State.ProcessedIds[uniqueId] = true

                    -- SE FOR O PRIMEIRO FETCH: apenas registra o ID, NÃO cria linha
                    if IsFirstFetch then
                        continue -- pula para o próximo, não exibe e não dá autojoin
                    end

                    -- A partir do segundo fetch: exibe normalmente os novos
                    if item.valor >= State.MinGenFilter then
                        itemCount = itemCount + 1

                        pcall(function()
                            createRow(item)
                            RunService.Heartbeat:Wait()
                            Scroll.CanvasPosition = Vector2.new(0, Scroll.CanvasSize.Y.Offset)
                        end)

                        setStatus("NEW: " .. item.nome .. " (" .. item.money .. ")", THEME.GreenActive)

                        if State.AutoJoin then
                            task.wait(0.3)
                            pcall(function()
                                TeleportService:TeleportToPlaceInstance(item.placeId or game.PlaceId, item.jobId, Player)
                            end)
                        end
                    end
                end
            end

            IsFirstFetch = false -- << após o primeiro ciclo, passa a mostrar novos

            if not StatusLabel.Text:find("NEW") then
                local t = os.date("%H:%M:%S")
                setStatus("Monitoring... " .. t, THEME.TextGray)
            end
        else
            failCount = failCount + 1
            connected = false
            setStatus("Reconnecting (" .. failCount .. ")", THEME.RedSoft)
            if failCount >= 5 then
                task.wait(2)
            end
        end
    end
end)

-- =============================================================================
-- ARRASTAR
-- =============================================================================
local function makeDraggable(obj, handle, others)
    local dragging, inputStart, startPos = false, nil, nil
    handle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            inputStart = input.Position
            startPos = obj.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
        and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local delta = input.Position - inputStart
        local newPos = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X,
                                 startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        obj.Position = newPos
        if others then
            for _, o in ipairs(others) do
                o.obj.Position = UDim2.new(newPos.X.Scale, newPos.X.Offset + o.offX,
                                           newPos.Y.Scale, newPos.Y.Offset + o.offY)
            end
        end
    end)
end

makeDraggable(MainFrame, DragHandle, {
    {obj = BorderShell,   offX = -4, offY = -4},
    {obj = DiscordButton, offX =  0, offY = 428},
})
makeDraggable(ToggleFrame, ToggleFrame)

print("[Canelloni] Joiner carregado! Aguardando novos dados do Firebase...")
