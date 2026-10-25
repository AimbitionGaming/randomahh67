print("[PF] Testing Matcha UI...")

if not UI or type(UI.AddTab) ~= "function" then
    warn("[PF] UI.AddTab is unavailable!")
else
    local ok, err = pcall(function()
        pcall(function()
            UI.RemoveTab("PF UI Test")
        end)

        UI.AddTab("PF UI Test", function(tab)
            local main = tab:Section("Settings", "Left")

            main:Toggle(
                "pf_test_enabled",
                "Enable Test",
                true
            )

            main:SliderInt(
                "pf_test_rpm",
                "Weapon RPM",
                100, 2500, 950
            )

            main:Combo(
                "pf_test_mode",
                "Fire Mode",
                {"Semi", "Burst", "Full Auto"},
                0
            )

            main:Text("Matcha UI is working!")
        end)
    end)

    if ok then
        print("[PF] UI registered successfully!")
    else
        warn("[PF] UI error:", err)
    end
end
