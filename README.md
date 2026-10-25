
-- Phantom Forces - Matcha
-- Clean Auto Fire + Anti-Recoil
-- UI: INS-UI (from your original script)
-- P: Open/close UI
-- F6: Enable/disable macro

-- Clean previous instances
if _G.PFOnlyStop then
    pcall(_G.PFOnlyStop)
end

if _G.MatchaTracerCleanup then
    pcall(_G.MatchaTracerCleanup)
end

if _G.MatchaTracerWindow then
    pcall(function()
        _G.MatchaTracerWindow:Destroy()
    end)
    _G.MatchaTracerWindow = nil
end

task.wait(0.2)

-- Load original UI library using Matcha-compatible HTTP
local url =
    "https://raw.githubusercontent.com/" ..
    "neaxusxgod-png/INS-ui/main/uilib.min.lua"

local loaded, loadErr = pcall(function()
    local source = httpget(url)
    local execute = loadstring(source)
    execute()
end)

local Lib = _G.INSui or _G.INSUI

if not Lib then
    warn("[PF] INS-UI failed to load:", loadErr)
    return
end

-- CONFIGURATION
local cfg = {
    enabled = false,
    autoFire = true,
    antiRecoil = true,

    trigger = "Hold RMB",
    mode = "Semi / Binary",

    rpm = 950,
    pressMs = 12,
    releaseMs = 30,
    burstTrim = 0,

    recoilStrength = 85,
    recoilX = 0,
    recoilY = 3
}

local state = {
    running = true,
    syntheticHeld = false,
    remainderX = 0,
    remainderY = 0
}

local win = nil

-- INPUT CONTROL
local function release()
    if state.syntheticHeld then
        state.syntheticHeld = false
        mouse1release()
    end
end

local function press()
    if not state.syntheticHeld then
        mouse1press()
        state.syntheticHeld = true
    end
end

local function triggerHeld()
    if cfg.trigger == "Hold RMB" then
        return ismouse2pressed()

    elseif cfg.trigger == "Hold F" then
        return iskeypressed(0x46)

    elseif cfg.trigger == "Hold LMB" then
        return ismouse1pressed()
    end

    return false
end

local function active()
    return state.running
        and cfg.enabled
        and isrbxactive()
        and triggerHeld()
end

local function intervalMs()
    return 60000 / math.max(1, cfg.rpm)
end

-- Interruptible timer
local function waitUntil(deadline)
    while state.running and tick() < deadline do
        if not active() then
            return false
        end
        task.wait(0.005)
    end

    return active()
end

local function waitMs(ms)
    return waitUntil(
        tick() + math.max(0, ms) / 1000
    )
end

-- RECOIL COMPENSATION
local function integerPart(n)
    if n >= 0 then
        return math.floor(n)
    end

    return math.ceil(n)
end

local function compensate()
    if not cfg.antiRecoil then return end
    if not active() then return end

    local strength = cfg.recoilStrength / 100

    state.remainderX = state.remainderX
        + cfg.recoilX * strength

    state.remainderY = state.remainderY
        + cfg.recoilY * strength

    local dx = integerPart(state.remainderX)
    local dy = integerPart(state.remainderY)

    state.remainderX = state.remainderX - dx
    state.remainderY = state.remainderY - dy

    if dx ~= 0 or dy ~= 0 then
        mousemoverel(dx, dy)
    end
end

-- SEMI-AUTO / BINARY
local function fireSemi()
    local interval = intervalMs()
    local start = tick()

    local hold = math.min(
        cfg.pressMs,
        math.max(5, interval - 5)
    )

    press()
    compensate()

    waitMs(hold)
    release()

    if active() then
        local elapsed = (tick() - start) * 1000

        local remaining = math.max(
            cfg.releaseMs,
            interval + 10 - elapsed
        )

        waitMs(remaining)
    end
end

-- HOLD-TO-BURST
local function fireBurst(rounds)
    local interval = intervalMs()

    -- Your requested formula
    local cycleMs =
        interval * rounds + 10

    local holdMs =
        interval * (rounds - 1)
        + 10 + cfg.burstTrim

    holdMs = math.max(
        5,
        math.min(holdMs, interval * rounds - 5)
    )

    local start = tick()
    local completed = true

    press()

    -- Apply recoil compensation for
    -- each theoretical bullet in the burst
    for i = 1, rounds do
        local target =
            start + ((i - 1) * interval) / 1000

        if not waitUntil(target) then
            completed = false
            break
        end

        compensate()
    end

    if completed then
        waitUntil(start + holdMs / 1000)
    end

    release()

    if active() then
        local elapsed = (tick() - start) * 1000

        local remaining = math.max(
            cfg.releaseMs,
            cycleMs - elapsed
        )

        waitMs(remaining)
    end
end

-- FULL AUTO
local function fireAuto()
    -- With physical LMB, allow normal
    -- continuous firing without synthetic presses.
    if cfg.trigger ~= "Hold LMB" then
        press()
    end

    while active()
        and cfg.autoFire
        and cfg.mode == "Full Auto" do

        compensate()

        if not waitMs(intervalMs()) then
            break
        end
    end

    release()
end

-- UNLOAD FUNCTION
_G.PFOnlyStop = function()
    state.running = false
    cfg.enabled = false
    release()

    if win then
        pcall(function()
            win:Destroy()
        end)
        win = nil
    end
end

-- UI
win = Lib:CreateWindow({
    title = "PF Fire Control",
    subtitle = "Auto Fire + Anti-Recoil",
    size = Vector2.new(580, 460),
    menuKey = "p",
    configName = "pf_fire_control",
    configFolder = "PFControl",
    autoSave = true,
    startOpen = true
})

local tab = win:Tab("Fire Control", "sliders")

local main = tab:Section("General", "Left")

main:Toggle("Enable Macro (F6)", false, function(v)
    cfg.enabled = v

    if not v then
        release()
    end
end)

main:Toggle("Auto Fire", true, function(v)
    cfg.autoFire = v
    if not v then release() end
end)

main:Toggle("Anti-Recoil", true, function(v)
    cfg.antiRecoil = v
end)

main:Dropdown(
    "Activation Button",
    {cfg.trigger},
    {"Hold RMB", "Hold F", "Hold LMB"},
    false,
    function(v)
        if v and v[1] then
            release()
            cfg.trigger = v[1]
        end
    end
)

main:Info(
    "Hold the activation button while Roblox is focused."
)

local fire = tab:Section("Fire Settings", "Left")

fire:Dropdown(
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

fire:Slider(
    "Weapon RPM",
    cfg.rpm, 10, 100, 2500, "RPM",
    function(v) cfg.rpm = v end
)

fire:Slider(
    "Semi Press Duration",
    cfg.pressMs, 1, 5, 50, "ms",
    function(v) cfg.pressMs = v end
)

fire:Slider(
    "Burst Hold Trim",
    cfg.burstTrim, 1, -20, 40, "ms",
    function(v) cfg.burstTrim = v end
)

fire:Slider(
    "Minimum Release",
    cfg.releaseMs, 5, 5, 200, "ms",
    function(v) cfg.releaseMs = v end
)

local recoil = tab:Section("Anti-Recoil", "Right")

recoil:Slider(
    "Compensation Strength",
    cfg.recoilStrength,
    5, 0, 150, "%",
    function(v) cfg.recoilStrength = v end
)

recoil:Slider(
    "Vertical Per Shot",
    cfg.recoilY,
    0.1, -15, 15, "px",
    function(v) cfg.recoilY = v end
)

recoil:Slider(
    "Horizontal Per Shot",
    cfg.recoilX,
    0.1, -15, 15, "px",
    function(v) cfg.recoilX = v end
)

recoil:Info(
    "Positive vertical moves down. " ..
    "Strength scales the configured mouse movement."
)

local tools = tab:Section("Controls", "Right")

tools:Button("Print Settings", function()
    print("[PF] Enabled:", cfg.enabled)
    print("[PF] Auto Fire:", cfg.autoFire)
    print("[PF] Recoil:", cfg.antiRecoil)
    print("[PF] Trigger:", cfg.trigger)
    print("[PF] Mode:", cfg.mode)
    print("[PF] RPM:", cfg.rpm)
    print("[PF] Shot interval:", intervalMs())
end)

tools:Button("Disable Macro", function()
    cfg.enabled = false
    release()
    print("[PF] Macro disabled")
end)

tools:Button("Unload", function()
    _G.PFOnlyStop()
end)

win:AddSettingsTab("cog")

-- FIRING ENGINE
task.spawn(function()
    while state.running do
        if not active() then
            release()
            state.remainderX = 0
            state.remainderY = 0

            task.wait(0.01)
        else
            if cfg.autoFire then
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
            else
                -- Recoil only during manual LMB firing
                if ismouse1pressed() then
                    compensate()
                end

                waitMs(intervalMs())
            end
        end
    end

    release()
end)

-- F6 HOTKEY
task.spawn(function()
    local previous = false

    while state.running do
        local pressed = iskeypressed(0x75)

        if pressed and not previous then
            cfg.enabled = not cfg.enabled

            if not cfg.enabled then
                release()
            end

            print(
                "[PF]",
                cfg.enabled and "ON" or "OFF"
            )
        end

        previous = pressed
        task.wait(0.03)
    end
end)

print("[PF] Fire Control loaded")
print("[PF] Press P to open UI")
print("[PF] F6 toggles macro")
