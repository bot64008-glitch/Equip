-- God Rock Smart Equip v6
-- Новое: ОБУЧЕНИЕ (скрипт запоминает, как ты экипируешь вручную), СКАН (отчёт в буфер),
-- вызов модулей игры, поиск не только Tool, более безопасные клики.

local ITEM_NAME = "God Rock"
local MODE = "item"        -- "item" или "slot"
local SLOT = 1             -- 1-10 (10 = клавиша 0)
local KEEP_ON = false
local DEBUG = true

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

local ui
local statusLabel
local itemBoxRef
local alive = true
local busy = false

local logBuf = {}
local function log(...)
    local n = select("#", ...)
    local parts = {}
    for i = 1, n do parts[i] = tostring((select(i, ...))) end
    local line = table.concat(parts, " ")
    table.insert(logBuf, line)
    if #logBuf > 80 then table.remove(logBuf, 1) end
    if DEBUG then print("[GodRock]", line) end
end
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

-- В этой игре предметы - Model (папка Tools), а не Tool, поэтому "в руках" определяем по нескольким признакам.
local childBase = {}
local function snapshotChildren()
    childBase = {}
    local ok, c = pcall(getChar)
    if not ok or not c then return end
    for _, v in ipairs(c:GetChildren()) do childBase[v] = true end
end
local function snapshotBaseline() snapshotChildren() end

-- новый прямой потомок персонажа (Model/Tool/Accessory) после экипировки
local function newChild()
    for _, v in ipairs(getChar():GetChildren()) do
        if not childBase[v] and (v:IsA("Tool") or v:IsA("Model") or v:IsA("Accessory")) then return v end
    end
end

local STORAGE_WORDS = {"tools", "bag", "pouch", "inventory", "backpack", "storage", "pets", "hotbar", "stash"}
local function inStorage(v, root)
    local x = v.Parent
    while x and x ~= root do
        local n = norm(x.Name)
        for _, w in ipairs(STORAGE_WORDS) do
            if n:find(w, 1, true) then return true end
        end
        x = x.Parent
    end
    return false
end

-- Model/Accessory/деталь с именем предмета вне папок-хранилищ
local function nonToolMatch()
    local c = getChar()
    for _, v in ipairs(c:GetDescendants()) do
        if match(v.Name) and (v:IsA("Model") or v:IsA("Accessory") or v:IsA("BasePart")) and not inStorage(v, c) then
            if not (v:IsA("BasePart") and v.Transparency >= 1) then return v end
        end
    end
end

-- атрибут / StringValue / ObjectValue с именем предмета вне хранилищ
local function valueMatch()
    local c = getChar()
    for _, val in pairs(c:GetAttributes()) do
        if type(val) == "string" and match(val) then return c end
    end
    for _, d in ipairs(c:GetDescendants()) do
        if d:IsA("StringValue") and d.Value ~= "" and match(d.Value) and not inStorage(d, c) then return d end
        if d:IsA("ObjectValue") and d.Value and match(d.Value.Name) and not inStorage(d, c) then return d end
    end
end

-- заголовок тулбара в GUI (показывает название предмета в руках)
local titleCache, titleT = {}, 0
local function titleSig()
    local pg = player:FindFirstChildOfClass("PlayerGui")
    if not pg then return "" end
    if os.clock() - titleT > 3 then
        titleT = os.clock()
        titleCache = {}
        for _, v in ipairs(pg:GetDescendants()) do
            if v:IsA("TextLabel") and foreign(v) and norm(v.Name) == "title"
                and norm(v:GetFullName()):find("toolbar", 1, true) then
                table.insert(titleCache, v)
            end
        end
    end
    local parts = {}
    for _, v in ipairs(titleCache) do
        if v.Parent then parts[#parts + 1] = v.Text end
    end
    return table.concat(parts, "|")
end

local slotToolName = nil
local lastAssumed = false
local preTitleSig = ""

local function held()
    if MODE == "slot" then
        local t = heldAny() or newChild()
        if t and (not slotToolName or t.Name == slotToolName) then return t end
        return nil
    end
    for _, v in ipairs(getChar():GetChildren()) do
        if v:IsA("Tool") and match(v.Name) then return v end
    end
    return nonToolMatch() or valueMatch()
end

-- для режима "Держать": заголовок тулбара тоже считается признаком
local function isHeldNow()
    if held() then return true end
    return MODE == "item" and match(titleSig())
end

-- проверка успеха после действия
local function verified()
    if held() then return true end
    local ts = titleSig()
    if ts ~= preTitleSig and ts ~= "" then
        if MODE == "slot" or match(ts) then return true end
    end
    return false
end

local function waitHeld(t)
    local t0 = os.clock()
    while os.clock() - t0 < (t or 0.6) do
        if verified() then return true end
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

-- ---------- клики ----------

local function relPath(inst, root)
    local names = {}
    local o = inst
    while o and o ~= root do
        table.insert(names, 1, o.Name)
        o = o.Parent
    end
    return names
end

local function resolvePath(names)
    local o = player:FindFirstChildOfClass("PlayerGui")
    for _, n in ipairs(names) do
        if not o then return nil end
        o = o:FindFirstChild(n)
    end
    return o
end

local clickPref = nil        -- "signal" или "real": что сработало раньше
local goodBtnPath = nil      -- путь к кнопке предмета, которая уже срабатывала
local goodSlotPaths = {}     -- то же для кнопок слотов

-- не нажимаем на магазины, обмены, яйца и т.п.
local BAD_WORDS = {"shop", "store", "buy", "purchase", "market", "trade", "sell", "gift",
    "robux", "crate", "egg", "auction", "gamepass", "codex", "craft", "mojo", "quest",
    "recipe", "upgrade", "enchant", "forge", "fuse", "combine", "claim"}
local function isBad(o)
    local x = o
    while x and not x:IsA("PlayerGui") do
        local n = norm(x.Name)
        for _, w in ipairs(BAD_WORDS) do
            if n:find(w, 1, true) then return true end
        end
        x = x.Parent
    end
    return false
end

local function isVisible(g)
    local o = g
    while o and not o:IsA("PlayerGui") do
        if o:IsA("GuiObject") and not o.Visible then return false end
        if o:IsA("LayerCollector") and not o.Enabled then return false end
        o = o.Parent
    end
    local cam = workspace.CurrentCamera
    local vp = cam and cam.ViewportSize or Vector2.new(2000, 2000)
    local p = g.AbsolutePosition + g.AbsoluteSize / 2
    return g.AbsoluteSize.X > 0 and g.AbsoluteSize.Y > 0 and p.X >= 0 and p.Y >= 0 and p.X <= vp.X and p.Y <= vp.Y
end

-- одиночный реальный клик по центру кнопки
local function clickOnce(btn)
    if not btn or not btn.Parent then return false end
    if btn.AbsoluteSize.X <= 0 then return false end
    local inset = GuiService:GetGuiInset()
    local p = btn.AbsolutePosition + btn.AbsoluteSize / 2 + inset
    local ok = pcall(function()
        VIM:SendMouseButtonEvent(p.X, p.Y, 0, true, game, 0)
        task.wait(0.05)
        VIM:SendMouseButtonEvent(p.X, p.Y, 0, false, game, 0)
    end)
    if not ok and typeof(firesignal) == "function" then
        pcall(function() firesignal(btn.Activated) end)
        pcall(function() firesignal(btn.MouseButton1Click) end)
    end
    return true
end

local function fireSignals(btn)
    for _, sig in ipairs({"MouseButton1Down", "MouseButton1Up", "MouseButton1Click", "Activated"}) do
        pcall(function() firesignal(btn[sig]) end)
    end
end

-- Нажатие кнопки ОДНИМ способом за раз и долгое ожидание результата
-- (двойное срабатывание у кастомных систем = экипировал и сразу снял).
local function activate(btn)
    local order = {"signal", "real"}
    if clickPref == "real" then order = {"real", "signal"} end
    for _, kind in ipairs(order) do
        local fired = false
        if kind == "signal" and typeof(firesignal) == "function" then
            fireSignals(btn)
            fired = true
        elseif kind == "real" then
            clickOnce(btn)
            fired = true
        end
        if fired then
            log("Клик способом:", kind)
            if waitHeld(1.8) then
                clickPref = kind
                return true
            end
        end
    end
    return false
end

local function collectButtons(test)
    local pg = player:FindFirstChildOfClass("PlayerGui")
    local list, seen = {}, {}
    if not pg then return list end
    for _, v in ipairs(pg:GetDescendants()) do
        if foreign(v) and test(v) then
            local b = clickable(v)
            if b and not seen[b] and isVisible(b) and not isBad(b) then
                seen[b] = true
                table.insert(list, b)
            end
        end
    end
    return list
end

-- если кнопки предмета не видно, пробуем открыть инвентарь
local INV_WORDS = {"inventory", "backpack", "bag", "items", "inv"}
local function openInventory()
    local btns = collectButtons(function(v)
        local n = norm(v.Name)
        local t = ""
        if v:IsA("TextLabel") or v:IsA("TextButton") then t = norm(v.Text) end
        for _, w in ipairs(INV_WORDS) do
            if n == w or t == w then return true end
        end
        return false
    end)
    local b = btns[1]
    if not b then return false end
    log("Открываю инвентарь:", b:GetFullName())
    clickOnce(b)
    task.wait(0.7)
    return true
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
    local function test(v)
        if (v:IsA("TextLabel") or v:IsA("TextButton")) and match(v.Text) then return true end
        return v:IsA("GuiObject") and match(v.Name)
    end
    -- 1) кнопка, которая уже срабатывала
    if goodBtnPath then
        local b = resolvePath(goodBtnPath)
        if b and b:IsA("GuiButton") and isVisible(b) then
            log("Кеш-кнопка:", b:GetFullName())
            if activate(b) then return true end
        end
    end
    -- 2) поиск заново (видимые кнопки, без магазинов)
    local cands = collectButtons(test)
    if #cands == 0 and openInventory() then cands = collectButtons(test) end
    for i, b in ipairs(cands) do
        if i > 4 or not alive then break end
        log("GUI-кнопка:", b:GetFullName())
        if activate(b) then
            goodBtnPath = relPath(b, pg)
            return true
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

local function slotTool()
    local bp = getBackpack()
    if not bp then return nil end
    local i = 0
    for _, v in ipairs(bp:GetChildren()) do
        if v:IsA("Tool") then
            i = i + 1
            if i == SLOT then return v end
        end
    end
end

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
    local function test(v)
        if (v:IsA("TextLabel") or v:IsA("TextButton")) and norm(v.Text) == num then return true end
        if v:IsA("GuiObject") then
            local n = norm(v.Name)
            return n == num or n == "slot" .. num or n == "hotbarslot" .. num or n == "item" .. num
        end
        return false
    end
    local cp = goodSlotPaths[SLOT]
    if cp then
        local b = resolvePath(cp)
        if b and b:IsA("GuiButton") and isVisible(b) then
            if activate(b) then return true end
        end
    end
    local cands = collectButtons(test)
    for i, b in ipairs(cands) do
        if i > 4 or not alive then break end
        log("Кнопка слота:", b:GetFullName())
        if activate(b) then
            goodSlotPaths[SLOT] = relPath(b, pg)
            return true
        end
    end
    return false
end

local function slot_remotes()
    local variants = {{SLOT}, {tostring(SLOT)}, {"Slot" .. SLOT}, {"Slot", SLOT}, {SLOT, true}}
    local t = slotTool()
    if t then table.insert(variants, 1, {t}) end
    return runRemotes(variants)
end

local function slot_backpack()
    local t = slotTool()
    if not t then return false end
    pcall(function() getHum():EquipTool(t) end)
    return waitHeld()
end

-- ---------- общие "умные" методы ----------

local function makeVariants()
    if MODE == "slot" then
        local v = {{SLOT}, {tostring(SLOT)}}
        local t = slotTool()
        if t then table.insert(v, 1, {t}) table.insert(v, {t.Name}) end
        return v
    end
    local v = {{ITEM_NAME}, {string.lower(ITEM_NAME)}, {ITEM_NAME, true}}
    local t = findTool()
    if t then table.insert(v, 1, {t}) end
    return v
end

local EQUIP_KEYS = {"equip", "equiptool", "equipitem", "equipslot", "hold", "holditem",
    "select", "selectslot", "selecttool", "useitem", "wield", "setequipped"}
local function isEquipKey(k)
    local n = norm(k)
    for _, w in ipairs(EQUIP_KEYS) do
        if n == w then return true end
    end
    return false
end

local function safeRequire(m)
    local res, done = nil, false
    task.spawn(function()
        local ok, r = pcall(require, m)
        if ok then res = r end
        done = true
    end)
    local t0 = os.clock()
    while not done and os.clock() - t0 < 1 do task.wait() end
    return res
end

local function callVariants(fn, selfObj, variants)
    for _, args in ipairs(variants) do
        if not alive then return false end
        if selfObj ~= nil then
            pcall(fn, selfObj, table.unpack(args))
            if waitHeld(0.3) then return true end
        end
        pcall(fn, table.unpack(args))
        if waitHeld(0.3) then return true end
    end
    return false
end

-- вызов функций Equip/Hold/Select из ModuleScript игры
local function method_modules()
    local variants = makeVariants()
    local roots = {RS}
    local ps = player:FindFirstChild("PlayerScripts")
    if ps then table.insert(roots, ps) end
    local pg = player:FindFirstChildOfClass("PlayerGui")
    if pg then table.insert(roots, pg) end
    local tried = 0
    for _, root in ipairs(roots) do
        for _, m in ipairs(root:GetDescendants()) do
            if m:IsA("ModuleScript") and foreign(m) and looksLikeEquip(m.Name) and tried < 25 then
                tried = tried + 1
                local mod = safeRequire(m)
                if type(mod) == "table" then
                    for k, f in pairs(mod) do
                        if type(f) == "function" and isEquipKey(k) then
                            log("Модуль:", m:GetFullName(), k)
                            if callVariants(f, mod, variants) then return true end
                        end
                    end
                end
            end
        end
    end
    return false
end

-- поиск таблиц с функциями Equip в памяти игры (нужен getgc)
local function method_gc()
    if typeof(getgc) ~= "function" then return false end
    local variants = makeVariants()
    local okg, gc = pcall(getgc, true)
    if not okg or type(gc) ~= "table" then return false end
    local found = {}
    for i, t in ipairs(gc) do
        if type(t) == "table" and #found < 40 then
            pcall(function()
                for k, f in pairs(t) do
                    if type(k) == "string" and type(f) == "function" and isEquipKey(k) then
                        table.insert(found, {t = t, f = f, k = k})
                        if #found >= 40 then break end
                    end
                end
            end)
        end
        if i % 2000 == 0 then task.wait() end
    end
    log("gc: найдено функций", #found)
    for _, it in ipairs(found) do
        if callVariants(it.f, it.t, variants) then
            log("gc сработал:", it.k)
            return true
        end
    end
    return false
end

-- ---------- умные ремоуты (только безопасные, с аргументами под игру) ----------

local function findItemModel()
    local c = getChar()
    for _, v in ipairs(c:GetDescendants()) do
        if (v:IsA("Model") or v:IsA("Tool")) and match(v.Name) then return v end
    end
    local wp = workspace:FindFirstChild("Players")
    local mine = wp and wp:FindFirstChild(player.Name)
    if mine then
        for _, v in ipairs(mine:GetDescendants()) do
            if (v:IsA("Model") or v:IsA("Tool")) and match(v.Name) then return v end
        end
    end
end

local cachedId, idSearched = nil, false
local function findItemId()
    if idSearched then return cachedId end
    idSearched = true
    local mods = RS:FindFirstChild("Modules")
    if not mods then return nil end
    for _, nm in ipairs({"ItemIDS", "ItemData"}) do
        local m = mods:FindFirstChild(nm)
        if m and m:IsA("ModuleScript") then
            local t = safeRequire(m)
            if type(t) == "table" then
                for k, v in pairs(t) do
                    if type(k) == "string" and match(k) and (type(v) == "number" or type(v) == "string") then
                        cachedId = v
                        log("ID предмета:", nm, k, v)
                        return cachedId
                    end
                    if type(k) == "number" and type(v) == "string" and match(v) then
                        cachedId = k
                        log("ID предмета:", nm, k, v)
                        return cachedId
                    end
                    if type(v) == "table" then
                        local n2 = v.Name or v.name or v.DisplayName
                        if type(n2) == "string" and match(n2) then
                            cachedId = v.ID or v.Id or v.id or k
                            log("ID предмета:", nm, tostring(cachedId))
                            return cachedId
                        end
                    end
                end
            end
        end
    end
    return nil
end

-- ТОЛЬКО эти ремоуты (Drop/Throw/Craft/Purchase/Swap и прочие не трогаем)
local EQUIP_REMOTE_NAMES = {"equiptool", "equipitem", "toolswapfrombag", "selecttool", "holdtool", "equipslot", "equip"}
local function equipRemoteRank(name)
    local n = norm(name)
    for i, w in ipairs(EQUIP_REMOTE_NAMES) do
        if n == w then return i end
    end
    return nil
end

local function smart_remotes()
    local model = findItemModel()
    local id = findItemId()
    local v = {}
    if MODE == "slot" then
        v = {{SLOT}, {tostring(SLOT)}, {SLOT, true}}
    else
        if model then table.insert(v, {model}) end
        table.insert(v, {ITEM_NAME})
        if id ~= nil then
            table.insert(v, {id})
            table.insert(v, {ITEM_NAME, id})
            table.insert(v, {id, ITEM_NAME})
        end
        if model and model.Name ~= ITEM_NAME then table.insert(v, {model.Name}) end
        table.insert(v, {SLOT, ITEM_NAME})
        table.insert(v, {ITEM_NAME, SLOT})
    end

    local list = {}
    for _, r in ipairs(RS:GetDescendants()) do
        if (r:IsA("RemoteEvent") or r:IsA("RemoteFunction")) then
            local rank = equipRemoteRank(r.Name)
            if rank then table.insert(list, {r = r, rank = rank}) end
        end
    end
    table.sort(list, function(a, b) return a.rank < b.rank end)
    log("Безопасных ремоутов:", #list, "модель:", model and model:GetFullName() or "нет", "id:", tostring(id))

    for _, it in ipairs(list) do
        for i, args in ipairs(v) do
            if not alive then return false end
            local r = it.r
            pcall(function()
                if r:IsA("RemoteEvent") then r:FireServer(table.unpack(args))
                else r:InvokeServer(table.unpack(args)) end
            end)
            if waitHeld(0.9) then
                log("Сработал ремоут:", r:GetFullName(), "вариант", i)
                return true
            end
        end
    end
    return false
end

-- ---------- ОБУЧЕНИЕ ----------

local learning = false
local learnCalls = {}
local learnClicks = {}
local learnKeys = {}
local learned = {remotes = {}, clicks = {}, keys = {}}
local hookOk = false

local function relPath(inst, root)
    local names = {}
    local o = inst
    while o and o ~= root do
        table.insert(names, 1, o.Name)
        o = o.Parent
    end
    return names
end

local function resolvePath(names)
    local o = player:FindFirstChildOfClass("PlayerGui")
    for _, n in ipairs(names) do
        if not o then return nil end
        o = o:FindFirstChild(n)
    end
    return o
end

local function ser(args)
    local parts = {}
    for i = 1, args.n do parts[#parts + 1] = tostring(args[i]) end
    return table.concat(parts, "|")
end

local function installHook()
    if typeof(hookmetamethod) ~= "function" or typeof(getnamecallmethod) ~= "function" then return false end
    local old
    local wrap = function(f) return f end
    if typeof(newcclosure) == "function" then wrap = newcclosure end
    local ok = pcall(function()
        old = hookmetamethod(game, "__namecall", wrap(function(self, ...)
            local m = getnamecallmethod()
            if learning and (m == "FireServer" or m == "InvokeServer") then
                local cn
                pcall(function() cn = self.ClassName end)
                if cn == "RemoteEvent" or cn == "RemoteFunction" then
                    table.insert(learnCalls, {remote = self, args = table.pack(...), t = os.clock()})
                end
            end
            return old(self, ...)
        end))
    end)
    return ok
end

local LEARN_SKIP = {"drop", "throw", "craft", "purchase", "buy", "sell", "trade", "unequip",
    "swing", "cosmetic", "music", "mouse", "camera", "move", "ping", "request"}
local function learnSkip(name)
    local n = norm(name)
    for _, w in ipairs(LEARN_SKIP) do
        if n:find(w, 1, true) then return true end
    end
    return false
end
local function learnWanted(name)
    local n = norm(name)
    for _, w in ipairs({"equip", "swap", "select", "hold", "retool", "target", "tool", "bag", "item"}) do
        if n:find(w, 1, true) then return true end
    end
    return false
end

local function fmtArg(v, depth)
    depth = depth or 0
    local t = typeof(v)
    if t == "string" then
        return '"' .. v .. '"'
    elseif t == "Instance" then
        return "<" .. v.ClassName .. " " .. v:GetFullName() .. ">"
    elseif t == "table" then
        if depth > 1 then return "{...}" end
        local parts, n = {}, 0
        for k, x in pairs(v) do
            n = n + 1
            if n > 6 then
                parts[#parts + 1] = "..."
                break
            end
            parts[#parts + 1] = tostring(k) .. "=" .. fmtArg(x, depth + 1)
        end
        return "{" .. table.concat(parts, ", ") .. "}"
    end
    return tostring(v)
end

local function fmtCall(c)
    local parts = {}
    for i = 1, c.args.n do parts[i] = fmtArg(c.args[i]) end
    return c.remote:GetFullName() .. "(" .. table.concat(parts, ", ") .. ")"
end

local function startLearn()
    if learning then return end
    if heldAny() or held() then
        status("Сними предмет вручную (как обычно)...")
        local w0 = os.clock()
        while alive and (heldAny() or held()) and os.clock() - w0 < 15 do task.wait(0.2) end
        task.wait(0.5)
    end
    snapshotChildren()
    local sig0 = titleSig()
    learnCalls, learnClicks, learnKeys = {}, {}, {}
    learning = true
    status("ОБУЧЕНИЕ: экипируй предмет вручную (20 сек)")

    local conn = UIS.InputBegan:Connect(function(input)
        local ut = input.UserInputType
        if ut == Enum.UserInputType.Touch or ut == Enum.UserInputType.MouseButton1 then
            local pg = player:FindFirstChildOfClass("PlayerGui")
            if pg then
                local inset = GuiService:GetGuiInset()
                for _, dy in ipairs({0, inset.Y}) do
                    local okg, objs = pcall(function()
                        return pg:GetGuiObjectsAtPosition(input.Position.X, input.Position.Y + dy)
                    end)
                    local done = false
                    if okg then
                        for _, o in ipairs(objs) do
                            if foreign(o) then
                                local b = clickable(o)
                                if b then
                                    table.insert(learnClicks, {inst = b, path = relPath(b, pg), t = os.clock()})
                                    done = true
                                    break
                                end
                            end
                        end
                    end
                    if done then break end
                end
            end
        elseif ut == Enum.UserInputType.Keyboard and not UIS:GetFocusedTextBox() then
            table.insert(learnKeys, {key = input.KeyCode, t = os.clock()})
        end
    end)

    local t0 = os.clock()
    local tEnd = nil
    while alive and os.clock() - t0 < 20 do
        task.wait(0.1)
        local gotCall = false
        for _, c in ipairs(learnCalls) do
            local okn, nm = pcall(function() return c.remote.Name end)
            if okn and learnWanted(nm) and not learnSkip(nm) then
                gotCall = true
                break
            end
        end
        local changed = (heldAny() ~= nil) or (newChild() ~= nil) or (titleSig() ~= sig0)
        if gotCall or (changed and (#learnClicks > 0 or #learnKeys > 0)) then
            tEnd = os.clock()
            break
        end
    end
    task.wait(1.0)
    conn:Disconnect()
    learning = false

    if not tEnd then
        status("Обучение: действий не заметил" .. (hookOk and "" or " (перехват ремоутов недоступен)"))
        return
    end

    learned = {remotes = {}, clicks = {}, keys = {}}
    local seenCall = {}
    for _, c in ipairs(learnCalls) do
        local okn, nm = pcall(function() return c.remote.Name end)
        if okn and c.t >= tEnd - 4 and learnWanted(nm) and not learnSkip(nm) then
            local key = tostring(c.remote) .. "#" .. ser(c.args)
            if not seenCall[key] and #learned.remotes < 6 then
                seenCall[key] = true
                table.insert(learned.remotes, {remote = c.remote, args = c.args})
            end
        end
    end
    for _, c in ipairs(learnClicks) do
        if c.t >= tEnd - 6 and #learned.clicks < 6 then
            table.insert(learned.clicks, c)
        end
    end
    for _, k in ipairs(learnKeys) do
        if k.t >= tEnd - 3 and #learned.keys < 4 then
            table.insert(learned.keys, k.key)
        end
    end

    local rep = {"=== LEARNED ==="}
    for _, c in ipairs(learned.remotes) do rep[#rep + 1] = "remote " .. fmtCall(c) end
    for _, c in ipairs(learned.clicks) do rep[#rep + 1] = "click " .. table.concat(c.path, ".") end
    for _, k in ipairs(learned.keys) do rep[#rep + 1] = "key " .. tostring(k) end
    local text = table.concat(rep, "\n")
    print(text)
    for _, l in ipairs(rep) do log(l) end
    local copied = false
    if typeof(setclipboard) == "function" then
        copied = pcall(setclipboard, text)
    end

    local msg = "Запомнил: ремоутов " .. #learned.remotes .. ", кликов " .. #learned.clicks .. ", клавиш " .. #learned.keys
    if copied then msg = msg .. " (в буфере)" end
    if not hookOk then msg = msg .. " (перехват ремоутов недоступен)" end
    status(msg)
end

local function learned_replay()
    if #learned.clicks == 0 and #learned.remotes == 0 and #learned.keys == 0 then return false end

    -- Повторяем ОДИН тип действий (ремоуты > клики > клавиши), чтобы не переключить дважды.
    if #learned.remotes > 0 then
        for _, c in ipairs(learned.remotes) do
            if not alive then return false end
            pcall(function()
                if c.remote.ClassName == "RemoteEvent" then
                    c.remote:FireServer(table.unpack(c.args, 1, c.args.n))
                else
                    c.remote:InvokeServer(table.unpack(c.args, 1, c.args.n))
                end
            end)
            task.wait(0.15)
        end
        if waitHeld(1.2) then return true end
        return true, true
    end

    if #learned.clicks > 0 then
        for _, c in ipairs(learned.clicks) do
            if not alive then return false end
            local b = (c.inst and c.inst.Parent) and c.inst or resolvePath(c.path)
            if b then
                clickOnce(b)
                task.wait(0.35)
            end
        end
        if waitHeld(1.2) then return true end
        return true, true
    end

    for _, k in ipairs(learned.keys) do
        tapKey(k)
        task.wait(0.15)
    end
    if waitHeld(1.2) then return true end
    return true, true
end

-- ---------- СКАН ----------

local function scan()
    local lines = {}
    local function add(s) lines[#lines + 1] = s end
    add("=== GodRock SCAN === режим=" .. MODE .. " предмет=" .. ITEM_NAME .. " слот=" .. SLOT)

    add("-- Character:")
    for _, v in ipairs(getChar():GetChildren()) do
        if v:IsA("Tool") or v:IsA("Model") or v:IsA("Accessory") or v:IsA("Folder") then
            add("  " .. v.ClassName .. " " .. v.Name)
        end
    end
    add("-- Backpack:")
    local bp = getBackpack()
    if bp then
        for _, v in ipairs(bp:GetChildren()) do add("  " .. v.ClassName .. " " .. v.Name) end
    end

    add("-- Совпадения по имени:")
    local n, seen = 0, 0
    for _, root in ipairs({player, RS, game:GetService("Workspace")}) do
        for _, v in ipairs(root:GetDescendants()) do
            seen = seen + 1
            if seen % 3000 == 0 then task.wait() end
            if n < 30 and foreign(v) and match(v.Name) then
                n = n + 1
                add("  " .. v.ClassName .. " " .. v:GetFullName())
            end
        end
    end

    local pg = player:FindFirstChildOfClass("PlayerGui")
    add("-- ScreenGui игры:")
    if pg then
        for _, g in ipairs(pg:GetChildren()) do
            if g:IsA("ScreenGui") and foreign(g) then
                add("  " .. g.Name .. (g.Enabled and "" or " (выкл)"))
            end
        end
        add("-- Текст/кнопки с именем предмета:")
        local c = 0
        for _, v in ipairs(pg:GetDescendants()) do
            if c < 15 and foreign(v) and ((v:IsA("TextLabel") or v:IsA("TextButton")) and match(v.Text)) then
                c = c + 1
                add("  " .. v.ClassName .. " " .. v:GetFullName())
            end
        end
    end

    add("-- Ремоуты (equip-подобные):")
    local rc = 0
    for _, root in ipairs({RS, game:GetService("Workspace"), player}) do
        for _, v in ipairs(root:GetDescendants()) do
            if rc < 40 and (v:IsA("RemoteEvent") or v:IsA("RemoteFunction")) and looksLikeEquip(v.Name) then
                rc = rc + 1
                add("  " .. v.ClassName .. " " .. v:GetFullName())
            end
        end
    end

    add("-- Модули (equip/inventory-подобные):")
    local mc = 0
    for _, root in ipairs({RS, player}) do
        for _, v in ipairs(root:GetDescendants()) do
            if mc < 20 and v:IsA("ModuleScript") and foreign(v) and looksLikeEquip(v.Name) then
                mc = mc + 1
                add("  " .. v:GetFullName())
            end
        end
    end

    add("-- Модель предмета / ID:")
    local mdl = findItemModel()
    add("  модель: " .. (mdl and mdl:GetFullName() or "нет") .. "  id: " .. tostring(findItemId()))
    add("-- held=" .. tostring(held() ~= nil) .. " title=[" .. titleSig() .. "]")
    add("-- Атрибуты персонажа:")
    local ac = 0
    for k, val in pairs(getChar():GetAttributes()) do
        ac = ac + 1
        if ac <= 15 then add("  " .. tostring(k) .. "=" .. tostring(val)) end
    end
    if pg then
        add("-- Тексты в тулбаре:")
        local tc = 0
        for _, v in ipairs(pg:GetDescendants()) do
            if tc < 25 and v:IsA("TextLabel") and foreign(v) and norm(v:GetFullName()):find("toolbar", 1, true) then
                tc = tc + 1
                add("  " .. v:GetFullName() .. " = [" .. v.Text .. "]")
            end
        end
    end
    add("-- Состояние: clickPref=" .. tostring(clickPref) .. " goodBtn=" .. (goodBtnPath and table.concat(goodBtnPath, ".") or "нет"))
    add("-- Последний лог:")
    for _, l in ipairs(logBuf) do add("  " .. l) end

    local report = table.concat(lines, "\n")
    print(report)
    if typeof(setclipboard) == "function" then
        pcall(setclipboard, report)
        status("Скан скопирован в буфер - вставь мне в чат")
    else
        status("Скан напечатан в консоль (F9)")
    end
end

-- ---------- запуск экипировки ----------

local function getMethods()
    if MODE == "slot" then
        return {
            {"обученное",      learned_replay},
            {"слот: клавиша",  slot_hotkey},
            {"слот: GUI",      slot_gui},
            {"слот: ремоуты",  smart_remotes},
            {"слот: Backpack", slot_backpack},
            {"модули",         method_modules},
            {"gc",             method_gc},
        }
    end
    return {
        {"обученное", learned_replay},
        {"GUI",            item_gui},
        {"умные ремоуты",  smart_remotes},
        {"EquipTool",      item_humanoid},
        {"модули",    method_modules},
        {"gc",        method_gc},
        {"клавиши",   item_hotkeys},
    }
end

local function doEquip()
    if held() then status("Уже в руках") return true end
    snapshotChildren()
    preTitleSig = titleSig()
    lastAssumed = false
    for _, m in ipairs(getMethods()) do
        if not alive then return false end
        status("Пробую: " .. m[1])
        local ok, res, assumed = pcall(m[2])
        if not ok then log("Ошибка метода", m[1], res) end
        if ok and res then
            if MODE == "slot" and not slotToolName then
                local t = heldAny() or newChild()
                if t then slotToolName = t.Name end
            end
            if assumed then
                lastAssumed = true
                status("Выполнено без проверки: " .. m[1])
            else
                status("Готово: " .. m[1])
            end
            return true
        end
    end
    status("Не вышло. Нажми ОБУЧЕНИЕ и экипируй вручную, или СКАН")
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

hookOk = installHook()
task.spawn(snapshotBaseline)
player.CharacterAdded:Connect(function()
    task.wait(1.5)
    snapshotBaseline()
end)

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

pcall(function()
    local p = guiParent()
    local old = p:FindFirstChild("GodRockEquipUI")
    if old then old:Destroy() end
end)

ui = make("ScreenGui", {Name = "GodRockEquipUI", ResetOnSpawn = false, DisplayOrder = 999, IgnoreGuiInset = true}, guiParent())
if not ui or not ui.Parent then error("меню не прикрепилось к экрану") end

local FULL_H = 282

local main = make("Frame", {
    Size = UDim2.fromOffset(230, FULL_H), Position = UDim2.new(0.5, -115, 0.04, 0),
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
    Position = UDim2.fromOffset(0, 30), Size = UDim2.new(1, 0, 0, FULL_H - 30), BackgroundTransparency = 1,
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
itemBoxRef = itemBox

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

local learnBtn = button("ОБУЧЕНИЕ", UDim2.fromOffset(10, 128),  UDim2.fromOffset(100, 30), body)
local scanBtn  = button("СКАН",     UDim2.fromOffset(120, 128), UDim2.fromOffset(100, 30), body)

local keepBtn = button("Держать: ВЫКЛ", UDim2.fromOffset(10, 166), UDim2.fromOffset(210, 30), body)

statusLabel = make("TextLabel", {
    Text = "Готов", Position = UDim2.fromOffset(10, 204), Size = UDim2.fromOffset(210, 44),
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

learnBtn.Activated:Connect(function()
    if busy then return end
    task.spawn(function()
        busy = true
        local ok, err = pcall(startLearn)
        if not ok then status("Ошибка обучения: " .. tostring(err)) end
        learning = false
        busy = false
    end)
end)

scanBtn.Activated:Connect(function()
    task.spawn(function()
        local ok, err = pcall(scan)
        if not ok then status("Ошибка скана: " .. tostring(err)) end
    end)
end)

keepBtn.Activated:Connect(function()
    KEEP_ON = not KEEP_ON
    refreshKeep()
    if KEEP_ON then task.spawn(equip) end
end)

local minimized = false
minBtn.Activated:Connect(function()
    minimized = not minimized
    body.Visible = not minimized
    main.Size = minimized and UDim2.fromOffset(230, 30) or UDim2.fromOffset(230, FULL_H)
    minBtn.Text = minimized and "+" or "-"
end)

local function shutdown()
    alive = false
    KEEP_ON = false
    learning = false
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

-- цикл "Держать" (после неудачи пауза, чтобы не спамить кликами)
task.spawn(function()
    local nextKeep = 0
    while alive do
        task.wait(1)
        if KEEP_ON and not busy and not lastAssumed and os.clock() >= nextKeep then
            local ok, h = pcall(isHeldNow)
            if ok and not h then
                local done = equip()
                if not done then nextKeep = os.clock() + 8 end
            end
        end
    end
end)

notify("God Rock Equip", "Меню загружено" .. (hookOk and "" or " (перехват ремоутов недоступен)"))
end

local ok, err = xpcall(main, function(e) return tostring(e) .. "\n" .. debug.traceback() end)
if not ok then
    warn("[GodRock] ОШИБКА: " .. tostring(err))
    notify("God Rock: ошибка", string.sub(tostring(err), 1, 200))
end
