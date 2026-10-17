-- God Rock Smart Equip v3 (с меню)
-- Меню: режим "Предмет" / "Слот", кнопка экипировки, режим "Держать", сворачивание и закрытие.

local ITEM_NAME = "God Rock"
local MODE = "item"        -- "item" или "slot"
local SLOT = 1             -- 1-10 (10 = клавиша 0)
local KEEP_ON = false
local DEBUG = true

-- Показ ошибок на экране (на телефоне консоль неудобна)
local function notify(title, text)
    pcall(function()
        game:GetService("StarterGui"):SetCore("SendNotification", {Title = title, Text = tostring(text), Duration = 8})
    end)
end

local function main()
local Players    = game:GetService("Players")
local RS         = game:GetService("ReplicatedStorage")
local GuiService = game:GetService("GuiService")
local UIS        = game:GetService("UserInputService")
local VIM        = game:GetService("VirtualInputManager")
local player     = Players.LocalPlayer

-- ============================ ЛОГИКА ============================

local ui            -- ScreenGui (заполняется ниже)
local statusLabel   -- строка статуса в меню
local alive = true

local function log(...) if DEBUG then print("[GodRock]", ...) end end
local function status(t)
    log(t)
    if statusLabel then statusLabel.Text = t end
end

local function norm(s) return string.lower((tostring(s):gsub("[%s_%-%.]", ""))) end
local TARGET = norm(ITEM_NAME)
local function match(s) return norm(s):find(TARGET, 1, true) ~= nil end
local function foreign(v) return not (ui and v:IsDescendantOf(ui)) end

local function getChar() return player.Character or player.CharacterAdded:Wait() end
local function getHum() return getChar():WaitForChild("Humanoid") end
local function getBackpack() return player:FindFirstChildOfClass("Backpack") end

local function heldAny()
    for _, v in ipairs(getChar():GetChildren()) do
        if v:IsA("Tool") then return v end
    end
end

local slotToolName = nil

local function held()
    local t = heldAny()
    if MODE == "slot" then
        if t and (not slotToolName or t.Name == slotToolName) then return t end
        return nil
    end
    for _, v in ipairs(getChar():GetChildren()) do
        if v:IsA("Tool") and match(v.Name) then return v end
    end
end

local function waitHeld(t)
    local t0 = os.clock()
    while os.clock() - t0 < (t or 0.6) do
        if held() then return true end
        task.wait(0.05)
    end
    return false
end

local REMOTE_WORDS = {"equip", "hold", "select", "tool", "item", "inventory", "backpack", "hotbar", "use", "wield", "slot"}
local function looksLikeEquip(name)
    name = string.lower(name)
    for _, w in ipairs(REMOTE_WORDS) do
        if name:find(w, 1, true) then return true end
    end
end

local function runRemotes(variants)
    local list = {}
    for _, root in ipairs({RS, game:GetService("Workspace"), player}) do
        for _, v in ipairs(root:GetDescendants()) do
            if (v:IsA("RemoteEvent") or v:IsA("RemoteFunction")) and looksLikeEquip(v.Name) then
                table.insert(list, v)
            end
        end
    end
    log("Кандидатов-ремоутов:", #list)
    for _, r in ipairs(list) do
        for _, args in ipairs(variants) do
            if not alive then return false end
            pcall(function()
                if r:IsA("RemoteEvent") then r:FireServer(table.unpack(args))
                else r:InvokeServer(table.unpack(args)) end
            end)
            if waitHeld(0.4) then
                log("Сработал ремоут:", r:GetFullName())
                return true
            end
        end
    end
    return false
end

local function clickable(obj)
    local o = obj
    while o and not o:IsA("LayerCollector") do
        if o:IsA("GuiButton") then return o end
        o = o.Parent
    end
end

local function activate(btn)
    if typeof(firesignal) == "function" then
        for _, sig in ipairs({"Activated", "MouseButton1Click", "MouseButton1Down", "MouseButton1Up"}) do
            pcall(function() firesignal(btn[sig]) end)
        end
    end
    if typeof(getconnections) == "function" then
        for _, sig in ipairs({"Activated", "MouseButton1Click"}) do
            pcall(function()
                for _, c in ipairs(getconnections(btn[sig])) do c:Fire() end
            end)
        end
    end
    if held() then return true end
    local inset = GuiService:GetGuiInset()
    local p = btn.AbsolutePosition + btn.AbsoluteSize / 2 + inset
    pcall(function()
        VIM:SendMouseButtonEvent(p.X, p.Y, 0, true, game, 0)
        task.wait(0.05)
        VIM:SendMouseButtonEvent(p.X, p.Y, 0, false, game, 0)
    end)
    return waitHeld(0.5)
end

local function tapKey(k)
    pcall(function()
        VIM:SendKeyEvent(true, k, false, game)
        task.wait(0.03)
        VIM:SendKeyEvent(false, k, false, game)
    end)
end

local SLOT_KEYS = {
    Enum.KeyCode.One, Enum.KeyCode.Two, Enum.KeyCode.Three, Enum.KeyCode.Four, Enum.KeyCode.Five,
    Enum.KeyCode.Six, Enum.KeyCode.Seven, Enum.KeyCode.Eight, Enum.KeyCode.Nine, Enum.KeyCode.Zero,
}

-- ---------- режим "item" ----------

local function findTool()
    local bp = getBackpack()
    if bp then
        for _, v in ipairs(bp:GetChildren()) do
            if v:IsA("Tool") and match(v.Name) then return v, "Backpack" end
        end
    end
    local pg = player:FindFirstChildOfClass("PlayerGui")
    for _, root in ipairs({player, getChar()}) do
        for _, v in ipairs(root:GetDescendants()) do
            if v:IsA("Tool") and match(v.Name) and not (pg and v:IsDescendantOf(pg)) then
                return v, v:GetFullName()
            end
        end
    end
end

local function item_humanoid()
    local tool, where = findTool()
    if not tool then return false end
    log("Tool найден:", where)
    pcall(function() getHum():EquipTool(tool) end)
    if waitHeld() then return true end
    pcall(function() tool.Parent = getBackpack() end)
    pcall(function() getHum():EquipTool(tool) end)
    return waitHeld()
end

local function item_remotes()
    local tool = findTool()
    local variants = {{ITEM_NAME}, {string.lower(ITEM_NAME)}, {ITEM_NAME, true}, {{Name = ITEM_NAME}}}
    if tool then table.insert(variants, 1, {tool}) end
    return runRemotes(variants)
end

local function item_gui()
    local pg = player:FindFirstChildOfClass("PlayerGui")
    if not pg then return false end
    local seen = {}
    for _, v in ipairs(pg:GetDescendants()) do
        if foreign(v) then
            local hit = false
            if (v:IsA("TextLabel") or v:IsA("TextButton")) and match(v.Text) then hit = true end
            if v:IsA("GuiObject") and match(v.Name) then hit = true end
            if hit then
                local btn = clickable(v)
                if btn and not seen[btn] and btn.AbsoluteSize.X > 0 then
                    seen[btn] = true
                    log("GUI-кнопка:", btn:GetFullName())
                    if activate(btn) then return true end
                end
            end
        end
    end
    return false
end

local function item_hotkeys()
    for i, k in ipairs(SLOT_KEYS) do
        if not alive then return false end
        tapKey(k)
        if waitHeld(0.3) then log("Сработал слот:", i) return true end
        if heldAny() then
            pcall(function() getHum():UnequipTools() end)
            task.wait(0.1)
        end
    end
    return false
end

-- ---------- режим "slot" ----------

local function slot_hotkey()
    local k = SLOT_KEYS[SLOT]
    if not k then return false end
    tapKey(k)
    return waitHeld(0.5)
end

local function slot_gui()
    local pg = player:FindFirstChildOfClass("PlayerGui")
    if not pg then return false end
    local num = tostring(SLOT)
    local seen = {}
    for _, v in ipairs(pg:GetDescendants()) do
        if foreign(v) then
            local hit = false
            if (v:IsA("TextLabel") or v:IsA("TextButton")) and norm(v.Text) == num then hit = true end
            if v:IsA("GuiObject") then
                local n = norm(v.Name)
                if n == num or n == "slot" .. num or n == "hotbarslot" .. num or n == "item" .. num then hit = true end
            end
            if hit then
                local btn = clickable(v)
                if btn and not seen[btn] and btn.AbsoluteSize.X > 0 then
                    seen[btn] = true
                    log("Кнопка слота:", btn:GetFullName())
                    if activate(btn) then return true end
                end
            end
        end
    end
    return false
end

local function slot_remotes()
    local variants = {{SLOT}, {tostring(SLOT)}, {"Slot" .. SLOT}, {"Slot", SLOT}, {SLOT, true}}
    local bp = getBackpack()
    if bp then
        local i = 0
        for _, v in ipairs(bp:GetChildren()) do
            if v:IsA("Tool") then
                i = i + 1
                if i == SLOT then table.insert(variants, 1, {v}) break end
            end
        end
    end
    return runRemotes(variants)
end

local function slot_backpack()
    local bp = getBackpack()
    if not bp then return false end
    local i = 0
    for _, v in ipairs(bp:GetChildren()) do
        if v:IsA("Tool") then
            i = i + 1
            if i == SLOT then
                pcall(function() getHum():EquipTool(v) end)
                return waitHeld()
            end
        end
    end
    return false
end

local function getMethods()
    if MODE == "slot" then
        return {
            {"слот: клавиша",  slot_hotkey},
            {"слот: GUI",      slot_gui},
            {"слот: ремоуты",  slot_remotes},
            {"слот: Backpack", slot_backpack},
        }
    end
    return {
        {"EquipTool", item_humanoid},
        {"ремоуты",   item_remotes},
        {"GUI",       item_gui},
        {"клавиши",   item_hotkeys},
    }
end

-- ---------- запуск экипировки ----------

local busy = false

local function doEquip()
    if held() then status("Уже в руках") return true end
    if heldAny() then
        pcall(function() getHum():UnequipTools() end)
        task.wait(0.15)
    end
    for _, m in ipairs(getMethods()) do
        if not alive then return false end
        status("Пробую: " .. m[1])
        local ok, res = pcall(m[2])
        if ok and res and held() then
            if MODE == "slot" and not slotToolName then
                slotToolName = heldAny().Name
            end
            status("Готово: " .. m[1])
            return true
        end
    end
    status("Не удалось экипировать")
    return false
end

local function equip()
    if busy then return false end
    busy = true
    local ok, res = pcall(doEquip)
    if not ok then status("Ошибка: " .. tostring(res)) end
    busy = false
    return ok and res
end

-- ============================ МЕНЮ ============================

local BLACK  = Color3.fromRGB(15, 15, 15)
local DARK   = Color3.fromRGB(30, 30, 30)
local GRAY   = Color3.fromRGB(70, 70, 70)
local LGRAY  = Color3.fromRGB(150, 150, 150)
local YELLOW = Color3.fromRGB(255, 200, 0)
local WHITE  = Color3.fromRGB(255, 255, 255)

local function guiParent()
    local pg = player:FindFirstChildOfClass("PlayerGui") or player:WaitForChild("PlayerGui", 10)
    if pg then return pg end
    if typeof(gethui) == "function" then
        local ok, h = pcall(gethui)
        if ok and h then return h end
    end
    return game:GetService("CoreGui")
end

local function make(class, props, parent)
    local o = Instance.new(class)
    for k, v in pairs(props) do o[k] = v end
    o.Parent = parent
    return o
end

local function corner(o, r) make("UICorner", {CornerRadius = UDim.new(0, r or 6)}, o) end

local function button(text, pos, size, parent)
    local b = make("TextButton", {
        Text = text, Position = pos, Size = size, BackgroundColor3 = GRAY, TextColor3 = WHITE,
        Font = Enum.Font.GothamBold, TextSize = 14, AutoButtonColor = false, BorderSizePixel = 0,
    }, parent)
    corner(b)
    return b
end

-- убираем старое меню при повторном запуске
pcall(function()
    local p = guiParent()
    local old = p:FindFirstChild("GodRockEquipUI")
    if old then old:Destroy() end
end)

ui = make("ScreenGui", {Name = "GodRockEquipUI", ResetOnSpawn = false, DisplayOrder = 999, IgnoreGuiInset = true}, guiParent())

if not ui or not ui.Parent then error("меню не прикрепилось к экрану") end

local main = make("Frame", {
    Size = UDim2.fromOffset(230, 236), Position = UDim2.new(0.5, -115, 0.25, 0),
    BackgroundColor3 = BLACK, BorderSizePixel = 0,
}, ui)
corner(main, 8)
make("UIStroke", {Color = YELLOW, Thickness = 2}, main)

local bar = make("Frame", {Size = UDim2.new(1, 0, 0, 30), BackgroundColor3 = DARK, BorderSizePixel = 0}, main)
corner(bar, 8)
make("TextLabel", {
    Text = "GOD ROCK EQUIP", Position = UDim2.fromOffset(10, 0), Size = UDim2.new(1, -80, 1, 0),
    BackgroundTransparency = 1, TextColor3 = YELLOW, Font = Enum.Font.GothamBold, TextSize = 14,
    TextXAlignment = Enum.TextXAlignment.Left,
}, bar)
local minBtn   = button("-", UDim2.new(1, -62, 0, 3), UDim2.fromOffset(26, 24), bar)
local closeBtn = button("X", UDim2.new(1, -32, 0, 3), UDim2.fromOffset(26, 24), bar)
closeBtn.BackgroundColor3 = Color3.fromRGB(170, 40, 40)

local body = make("Frame", {
    Position = UDim2.fromOffset(0, 30), Size = UDim2.new(1, 0, 0, 206), BackgroundTransparency = 1,
}, main)

local modeItem = button("ПРЕДМЕТ", UDim2.fromOffset(10, 8),   UDim2.fromOffset(100, 30), body)
local modeSlot = button("СЛОТ",    UDim2.fromOffset(120, 8),  UDim2.fromOffset(100, 30), body)

local itemBox = make("TextBox", {
    Text = ITEM_NAME, PlaceholderText = "Имя предмета", Position = UDim2.fromOffset(10, 46),
    Size = UDim2.fromOffset(210, 30), BackgroundColor3 = DARK, TextColor3 = WHITE,
    PlaceholderColor3 = LGRAY, Font = Enum.Font.Gotham, TextSize = 14, ClearTextOnFocus = false,
    BorderSizePixel = 0,
}, body)
corner(itemBox)

local slotRow = make("Frame", {Position = UDim2.fromOffset(10, 46), Size = UDim2.fromOffset(210, 30), BackgroundTransparency = 1}, body)
local slotMinus = button("-", UDim2.fromOffset(0, 0),   UDim2.fromOffset(40, 30), slotRow)
local slotLabel = make("TextLabel", {
    Text = "Слот " .. SLOT, Position = UDim2.fromOffset(50, 0), Size = UDim2.fromOffset(110, 30),
    BackgroundColor3 = DARK, TextColor3 = YELLOW, Font = Enum.Font.GothamBold, TextSize = 14, BorderSizePixel = 0,
}, slotRow)
corner(slotLabel)
local slotPlus = button("+", UDim2.fromOffset(170, 0), UDim2.fromOffset(40, 30), slotRow)

local equipBtn = button("ЭКИПИРОВАТЬ", UDim2.fromOffset(10, 84), UDim2.fromOffset(210, 36), body)
equipBtn.BackgroundColor3 = YELLOW
equipBtn.TextColor3 = BLACK
equipBtn.TextSize = 15

local keepBtn = button("Держать: ВЫКЛ", UDim2.fromOffset(10, 128), UDim2.fromOffset(210, 30), body)

statusLabel = make("TextLabel", {
    Text = "Готов", Position = UDim2.fromOffset(10, 166), Size = UDim2.fromOffset(210, 34),
    BackgroundTransparency = 1, TextColor3 = LGRAY, Font = Enum.Font.Gotham, TextSize = 12,
    TextWrapped = true, TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top,
}, body)

local function refreshMode()
    local isItem = MODE == "item"
    modeItem.BackgroundColor3 = isItem and YELLOW or GRAY
    modeItem.TextColor3 = isItem and BLACK or WHITE
    modeSlot.BackgroundColor3 = isItem and GRAY or YELLOW
    modeSlot.TextColor3 = isItem and WHITE or BLACK
    itemBox.Visible = isItem
    slotRow.Visible = not isItem
    slotToolName = nil
end
local function refreshKeep()
    keepBtn.Text = "Держать: " .. (KEEP_ON and "ВКЛ" or "ВЫКЛ")
    keepBtn.BackgroundColor3 = KEEP_ON and YELLOW or GRAY
    keepBtn.TextColor3 = KEEP_ON and BLACK or WHITE
end
refreshMode()
refreshKeep()

local conns = {}

modeItem.Activated:Connect(function() MODE = "item" refreshMode() end)
modeSlot.Activated:Connect(function() MODE = "slot" refreshMode() end)

itemBox.FocusLost:Connect(function()
    local t = itemBox.Text
    if t ~= "" then ITEM_NAME = t TARGET = norm(t) end
    slotToolName = nil
end)

slotMinus.Activated:Connect(function()
    SLOT = SLOT - 1
    if SLOT < 1 then SLOT = 10 end
    slotLabel.Text = "Слот " .. SLOT
    slotToolName = nil
end)
slotPlus.Activated:Connect(function()
    SLOT = SLOT + 1
    if SLOT > 10 then SLOT = 1 end
    slotLabel.Text = "Слот " .. SLOT
    slotToolName = nil
end)

equipBtn.Activated:Connect(function() task.spawn(equip) end)

keepBtn.Activated:Connect(function()
    KEEP_ON = not KEEP_ON
    refreshKeep()
    if KEEP_ON then task.spawn(equip) end
end)

local minimized = false
minBtn.Activated:Connect(function()
    minimized = not minimized
    body.Visible = not minimized
    main.Size = minimized and UDim2.fromOffset(230, 30) or UDim2.fromOffset(230, 236)
    minBtn.Text = minimized and "+" or "-"
end)

local function shutdown()
    alive = false
    KEEP_ON = false
    for _, c in ipairs(conns) do pcall(function() c:Disconnect() end) end
    pcall(function() ui:Destroy() end)
end
closeBtn.Activated:Connect(shutdown)

-- перетаскивание за заголовок (мышь и тач)
local dragging, dragStart, startPos
bar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = main.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
table.insert(conns, UIS.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local d = input.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
    end
end))

-- цикл "Держать"
task.spawn(function()
    while alive do
        task.wait(1)
        if KEEP_ON and not busy then
            local ok, h = pcall(held)
            if ok and not h then equip() end
        end
    end
end)

    notify("God Rock Equip", "Меню загружено")
end

local ok, err = xpcall(main, function(e) return tostring(e) .. "\n" .. debug.traceback() end)
if not ok then
    warn("[GodRock] ОШИБКА: " .. tostring(err))
    notify("God Rock: ошибка", string.sub(tostring(err), 1, 200))
end
