-- God Rock Smart Equip v2
-- Два режима:
--   "item" = экипирует предмет по имени (ITEM_NAME)
--   "slot" = экипирует предмет из слота хотбара (SLOT)

local MODE = "item"             -- "item" или "slot"
local ITEM_NAME = "God Rock"    -- для режима "item"
local SLOT = 1                  -- для режима "slot" (1-10, где 10 = клавиша 0)
local KEEP_EQUIPPED = true      -- пере-экипировать, если предмет пропал из рук
local DEBUG = true

local Players    = game:GetService("Players")
local RS         = game:GetService("ReplicatedStorage")
local GuiService = game:GetService("GuiService")
local VIM        = game:GetService("VirtualInputManager")
local player     = Players.LocalPlayer

local function log(...) if DEBUG then print("[GodRock]", ...) end end

local function norm(s)
    return string.lower((tostring(s):gsub("[%s_%-%.]", "")))
end
local TARGET = norm(ITEM_NAME)
local function match(s) return norm(s):find(TARGET, 1, true) ~= nil end

local function getChar() return player.Character or player.CharacterAdded:Wait() end
local function getHum() return getChar():WaitForChild("Humanoid") end
local function getBackpack() return player:FindFirstChildOfClass("Backpack") end

local function heldAny()
    for _, v in ipairs(getChar():GetChildren()) do
        if v:IsA("Tool") then return v end
    end
end

-- В режиме "slot" запоминаем, какой предмет оказался в слоте
local slotToolName = nil

-- Нужный предмет в руках?
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

-- Общие помощники ---------------------------------------------------------

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

-- РЕЖИМ "item" -------------------------------------------------------------

local function findTool()
    local bp = getBackpack()
    if bp then
        for _, v in ipairs(bp:GetChildren()) do
            if v:IsA("Tool") and match(v.Name) then return v, "Backpack" end
        end
    end
    for _, root in ipairs({player, getChar()}) do
        for _, v in ipairs(root:GetDescendants()) do
            if v:IsA("Tool") and match(v.Name) and not v:IsDescendantOf(player:FindFirstChildOfClass("PlayerGui") or game) then
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
    local variants = {
        {ITEM_NAME}, {string.lower(ITEM_NAME)}, {ITEM_NAME, true}, {{Name = ITEM_NAME}},
    }
    if tool then table.insert(variants, 1, {tool}) end
    return runRemotes(variants)
end

local function item_gui()
    local pg = player:FindFirstChildOfClass("PlayerGui")
    if not pg then return false end
    local seen = {}
    for _, v in ipairs(pg:GetDescendants()) do
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
    return false
end

local SLOT_KEYS = {
    Enum.KeyCode.One, Enum.KeyCode.Two, Enum.KeyCode.Three, Enum.KeyCode.Four, Enum.KeyCode.Five,
    Enum.KeyCode.Six, Enum.KeyCode.Seven, Enum.KeyCode.Eight, Enum.KeyCode.Nine, Enum.KeyCode.Zero,
}

local function item_hotkeys()
    for i, k in ipairs(SLOT_KEYS) do
        tapKey(k)
        if waitHeld(0.3) then log("Сработал слот:", i) return true end
        if heldAny() then
            pcall(function() getHum():UnequipTools() end)
            task.wait(0.1)
        end
    end
    return false
end

-- РЕЖИМ "slot" -------------------------------------------------------------

-- 1) Нажатие клавиши слота (работает и для стандартного, и для большинства кастомных хотбаров)
local function slot_hotkey()
    local k = SLOT_KEYS[SLOT]
    if not k then return false end
    tapKey(k)
    return waitHeld(0.5)
end

-- 2) Клик по кнопке слота в GUI (ищем кнопку с номером слота)
local function slot_gui()
    local pg = player:FindFirstChildOfClass("PlayerGui")
    if not pg then return false end
    local num = tostring(SLOT)
    local seen = {}
    for _, v in ipairs(pg:GetDescendants()) do
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
    return false
end

-- 3) Ремоуты со слотом / предметом по порядку
local function slot_remotes()
    local variants = {
        {SLOT}, {tostring(SLOT)}, {"Slot" .. SLOT}, {"Slot", SLOT}, {SLOT, true},
    }
    local bp = getBackpack()
    if bp then
        local i = 0
        for _, v in ipairs(bp:GetChildren()) do
            if v:IsA("Tool") then
                i += 1
                if i == SLOT then table.insert(variants, 1, {v}) break end
            end
        end
    end
    return runRemotes(variants)
end

-- 4) Запасной вариант: N-й Tool в Backpack (порядок может не совпадать с хотбаром)
local function slot_backpack()
    local bp = getBackpack()
    if not bp then return false end
    local i = 0
    for _, v in ipairs(bp:GetChildren()) do
        if v:IsA("Tool") then
            i += 1
            if i == SLOT then
                pcall(function() getHum():EquipTool(v) end)
                return waitHeld()
            end
        end
    end
    return false
end

-- Выбор методов -------------------------------------------------------------

local methods
if MODE == "slot" then
    methods = {
        {"Slot: hotkey",   slot_hotkey},
        {"Slot: GUI",      slot_gui},
        {"Slot: remotes",  slot_remotes},
        {"Slot: backpack", slot_backpack},
    }
else
    methods = {
        {"Humanoid:EquipTool", item_humanoid},
        {"Remotes",            item_remotes},
        {"GUI",                item_gui},
        {"Hotkeys",            item_hotkeys},
    }
end

local function equip()
    if held() then return true end

    -- в руках не тот предмет (или первый запуск в режиме слота) — сначала убираем
    if heldAny() then
        pcall(function() getHum():UnequipTools() end)
        task.wait(0.15)
    end

    for _, m in ipairs(methods) do
        log("Пробую:", m[1])
        local ok, res = pcall(m[2])
        if ok and res and held() then
            if MODE == "slot" and not slotToolName then
                slotToolName = heldAny().Name
                log("В слоте", SLOT, "предмет:", slotToolName)
            end
            log("Готово через", m[1])
            return true
        end
    end
    log("Не удалось экипировать", MODE == "slot" and ("слот " .. SLOT) or ITEM_NAME)
    return false
end

equip()

if KEEP_EQUIPPED then
    task.spawn(function()
        while task.wait(1) do
            if not held() then equip() end
        end
    end)
end
