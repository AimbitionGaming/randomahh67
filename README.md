
-- PF Matcha - Fire Control + GC Scaling
-- P = UI | F6 = Toggle
-- No mouse movement for recoil

if _G.PF_Stop then
    pcall(_G.PF_Stop)
end

local cfg = {
    enabled = false,
    autoFire = true,
    antiRecoil = true,
    reduction = 85,
    trigger = "LMB",
    mode = "Semi / Binary",
    rpm = 950,
    pressMs = 15,
    trimMs = 0,
    releaseMs = 40
}

local state = {
    running = true,
    held = false,
    physicalLMB = false,
    useEvents = false,
    suppressUntil = 0,
    connections = {},
    entries = {},
    window = nil
}

local recoilKeys = {
    "_firemodeStability",
    "hipfirespread",
    "aimfirespread"
}

local function release()
    if state.held then
        state.suppressUntil = tick() + 0.025
        mouse1release()
        state.held = false
    end
end

local function restoreRecoil()
    for _, item in ipairs(state.entries) do
        pcall(function()
            applygc(
                {item.entry},
                item.key,
                item.original
            )
        end)
    end
end

local function applyRecoil()
    if not cfg.antiRecoil then
        restoreRecoil()
        return
    end

    local factor =
        1 - math.max(0, math.min(
            100, cfg.reduction
        )) / 100

    local changed = 0

    for _, item in ipairs(state.entries) do
        local ok, count = pcall(function()
            return applygc(
                {item.entry},
                item.key,
                item.original * factor
            )
        end)

        if ok and type(count) == "number" then
            changed = changed + count
        end
    end

    return changed
end

local function scanRecoil()
    restoreRecoil()
    state.entries = {}

    local ok, results = pcall(function()
        return getgc(recoilKeys)
    end)

    if not ok or type(results) ~= "table" then
        warn("[PF] GC scan failed")
        return
    end

    for _, entry in ipairs(results) do
        if type(entry) == "table"
            and type(entry.value) == "number"
            and type(entry.key) == "string" then

            table.insert(state.entries, {
                entry = entry,
                key = entry.key,
                original = entry.value
            })
        end
    end

    local changed = applyRecoil()

    print(
        "[PF] GC entries:",
        #state.entries,
        "Modified:",
        changed or 0
    )
end

-- Physical LMB event tracking
local input = game:GetService("UserInputService")

local okEvents = pcall(function()
    local began = input.InputBegan:Connect(function(obj)
        if obj.UserInputType ==
            Enum.UserInputType.MouseButton1 then

            if tick() >= state.suppressUntil then
                state.physicalLMB = true
            end
        end
    end)

    local ended = input.InputEnded:Connect(function(obj)
        if obj.UserInputType ==
            Enum.UserInputType.MouseButton1 then

            if tick() >= state.suppressUntil then
                state.physicalLMB = false
            end
        end
    end)

    table.insert(state.connections, began)
    table.insert(state.connections, ended)
end)

state.useEvents = okEvents

local function triggerHeld()
    if cfg.trigger == "RMB" then
        return ismouse2pressed()
    elseif cfg.trigger == "F" then
        return iskeypressed(0x46)
    else
        if state.useEvents then
            return state.physicalLMB
        end
        return ismouse1pressed()
    end
end

local function active()
    return state.running
        and cfg.enabled
        and isrbxactive()
        and triggerHeld()
end

local function interval()
    return 60000 / math.max(1, cfg.rpm)
end

local function waitUntil(deadline)
    while state.running and tick() < deadline do
        if not active() then return false end
        task.wait(0.005)
    end
    return active()
end

local function waitMs(ms)
    return waitUntil(tick() + ms / 1000)
end

local function press()
    if not state.held then
        state.suppressUntil = tick() + 0.025
        mouse1press()
        state.held = true
    end
end

local function fireSemi()
    local spacing = interval()
    local started = tick()

    press()

    local hold = math.min(
        cfg.pressMs,
        math.max(5, spacing - 5)
    )

    waitMs(hold)
    release()

    if active() then
        local elapsed =
            (tick() - started) * 1000

        waitMs(math.max(
            cfg.releaseMs,
            spacing + 10 - elapsed
        ))
    end
end

local function fireBurst(rounds)
    local spacing = interval()

    local cycle = spacing * rounds + 10

    local hold =
        (rounds - 1) * spacing
        + 10 + cfg.trimMs

    hold = math.max(
        5,
        math.min(hold, rounds * spacing - 5)
    )

    local started = tick()

    press()
    waitUntil(started + hold / 1000)
    release()

    if active() then
        local elapsed =
            (tick() - started) * 1000

        waitMs(math.max(
            cfg.releaseMs,
            cycle - elapsed
        ))
    end
end

local function fireAuto()
    -- Physical LMB already maintains the hold.
    if cfg.trigger ~= "LMB" then
        press()
    end

    while active()
        and cfg.autoFire
        and cfg.mode == "Full Auto" do
        task.wait(0.01)
    end

    release()
end

_G.PF_Stop = function()
    cfg.enabled = false
    state.running = false
    release()
    restoreRecoil()

    for _, conn in ipairs(state.connections) do
        pcall(function()
            conn:Disconnect()
        end)
    end

    if state.window then
        pcall(function()
            state.window:Destroy()
        end)
    end
end

-- LOAD INS-UI
local url =
    "https://raw.githubusercontent.com/" ..
    "neaxusxgod-png/INS-ui/main/uilib.min.lua"

local loaded, loadError = pcall(function()
    loadstring(httpget(url))()
end)

local Lib = _G.INSui or _G.INSUI

if not loaded or not Lib then
    warn("[PF] UI load failed:", loadError)
    _G.PF_Stop()
    return
end

local win = Lib:CreateWindow({
    title = "PF Fire Control",
    subtitle = "Auto Fire + GC Compensation",
    size = Vector2.new(580, 450),
    menuKey = "p",
    configName = "pf_gc_fire",
    configFolder = "PFControl",
    autoSave = true,
    startOpen = true
})

state.window = win

local tab = win:Tab("Fire Control", "sliders")
local general = tab:Section("General", "Left")

general:Toggle("Enabled", false, function(v)
    cfg.enabled = v
    if not v then release() end
end)

general:Toggle("Auto Fire", true, function(v)
    cfg.autoFire = v
    if not v then release() end
end)

general:Toggle("GC Compensation", true, function(v)
    cfg.antiRecoil = v
    applyRecoil()
end)

general:Dropdown(
    "Activation",
    {cfg.trigger},
    {"LMB", "RMB", "F"},
    false,
    function(v)
        if v and v[1] then
            release()
            cfg.trigger = v[1]
        end
    end
)

local firing = tab:Section("Firing", "Left")

firing:Dropdown(
    "Fire Mode",
    {cfg.mode},
    {
        "Semi / Binary",
        "2 Round Burst",
        "3 Round Burst",
        "4 Round Burst",
        "Full Auto"
    },
    false,
    function(v)
        if v and v[1] then
            release()
            cfg.mode = v[1]
        end
    end
)

firing:Slider(
    "RPM",
    cfg.rpm, 10, 100, 2500, "RPM",
    function(v) cfg.rpm = v end
)

firing:Slider(
    "Press Duration",
    cfg.pressMs, 1, 5, 50, "ms",
    function(v) cfg.pressMs = v end
)

firing:Slider(
    "Burst Hold Trim",
    cfg.trimMs, 1, -20, 40, "ms",
    function(v) cfg.trimMs = v end
)

firing:Slider(
    "Minimum Release",
    cfg.releaseMs, 5, 5, 200, "ms",
    function(v) cfg.releaseMs = v end
)

local recoil = tab:Section("GC Settings", "Right")

recoil:Slider(
    "Value Reduction",
    cfg.reduction, 1, 0, 100, "%",
    function(v)
        cfg.reduction = v
        applyRecoil()
    end
)

recoil:Info(
    "Scales original stability/spread values. " ..
    "Actual camera recoil reduction is unverified."
)

recoil:Button("Rescan Weapon Values", function()
    scanRecoil()
end)

recoil:Button("Restore Original Values", function()
    restoreRecoil()
    cfg.antiRecoil = false
end)

local controls = tab:Section("Controls", "Right")

controls:Button("Diagnostics", function()
    print("[PF] Enabled:", cfg.enabled)
    print("[PF] Trigger:", cfg.trigger)
    print("[PF] Trigger held:", triggerHeld())
    print("[PF] Mode:", cfg.mode)
    print("[PF] RPM:", cfg.rpm)
    print("[PF] GC entries:", #state.entries)
    print("[PF] Event hook:", state.useEvents)
end)

controls:Button("Unload", function()
    _G.PF_Stop()
end)

win:AddSettingsTab("cog")

-- FIRING LOOP
task.spawn(function()
    while state.running do
        if not active() or not cfg.autoFire then
            release()
            task.wait(0.01)
        else
            if cfg.mode == "Semi / Binary" then
                fireSemi()
            elseif cfg.mode == "2 Round Burst" then
                fireBurst(2)
            elseif cfg.mode == "3 Round Burst" then
                fireBurst(3)
            elseif cfg.mode == "4 Round Burst" then
                fireBurst(4)
            elseif cfg.mode == "Full Auto" then
                fireAuto()
            end
        end
    end
    release()
end)

-- F6 HOTKEY
task.spawn(function()
    local previous = false

    while state.running do
        local down = iskeypressed(0x75)

        if down and not previous then
            cfg.enabled = not cfg.enabled
            if not cfg.enabled then release() end
            print("[PF] Enabled:", cfg.enabled)
        end

        previous = down
        task.wait(0.03)
    end
end)

-- Apply GC values on startup
task.spawn(function()
    task.wait(0.5)
    if state.running then
        scanRecoil()
    end
end)

print("[PF] Loaded. Press P for UI.")
