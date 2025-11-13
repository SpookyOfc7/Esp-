-- ESP pronto para loadstring (sem skeleton)

spawn(function()
    local Players = game:GetService("Players")
    local RunService = game:GetService("RunService")
    local LocalPlayer = Players.LocalPlayer
    local Camera = workspace.CurrentCamera

    local ESPSettings = {
        Enabled = true,
        Line = true,
        Box = true,
        Name = true,
        HealthBar = true
    }

    local ESPObjects = {}

    local function clamp(v, a, b) return math.max(a, math.min(b, v)) end

    local function RainbowColor(t)
        local r = math.sin(t * 2) * 0.5 + 0.5
        local g = math.sin(t * 2 + 2) * 0.5 + 0.5
        local b = math.sin(t * 2 + 4) * 0.5 + 0.5
        return Color3.new(r, g, b)
    end

    local function CreateESP(player)
        if player == LocalPlayer then return end
        if ESPObjects[player] then return end

        local esp = {}

        if ESPSettings.Line then
            esp.Line = Drawing.new("Line")
            esp.Line.Thickness = 1
            esp.Line.Visible = false
        end

        if ESPSettings.Box then
            esp.Box = Drawing.new("Square")
            esp.Box.Thickness = 2
            esp.Box.Filled = false
            esp.Box.Visible = false
        end

        if ESPSettings.Name then
            esp.Name = Drawing.new("Text")
            esp.Name.Size = 16
            esp.Name.Center = true
            esp.Name.Outline = true
            esp.Name.Color = Color3.fromRGB(255,255,255)
            esp.Name.Visible = false
        end

        if ESPSettings.HealthBar then
            esp.HealthOutline = Drawing.new("Line")
            esp.HealthOutline.Thickness = 3
            esp.HealthOutline.Color = Color3.new(0,0,0)
            esp.HealthOutline.Visible = false

            esp.HealthFill = Drawing.new("Line")
            esp.HealthFill.Thickness = 2
            esp.HealthFill.Visible = false
        end

        ESPObjects[player] = esp
    end

    local function RemoveESP(player)
        local esp = ESPObjects[player]
        if not esp then return end
        for _, v in pairs(esp) do
            if type(v) == "table" then
                for _, o in pairs(v) do
                    pcall(function()
                        if o and o.Remove then o:Remove() elseif o and o.Visible ~= nil then o.Visible = false end
                    end)
                end
            else
                pcall(function()
                    if v and v.Remove then v:Remove() elseif v and v.Visible ~= nil then v.Visible = false end
                end)
            end
        end
        ESPObjects[player] = nil
    end

    local function HideAll(esp)
        for _, v in pairs(esp) do
            if type(v) == "table" then
                for _, o in pairs(v) do
                    if o and o.Visible ~= nil then pcall(function() o.Visible = false end) end
                end
            else
                if v and v.Visible ~= nil then pcall(function() v.Visible = false end) end
            end
        end
    end

    for _, pl in pairs(Players:GetPlayers()) do CreateESP(pl) end
    Players.PlayerAdded:Connect(CreateESP)
    Players.PlayerRemoving:Connect(RemoveESP)

    local function ToScreen(pos)
        local v, onScreen = Camera:WorldToViewportPoint(pos)
        return Vector2.new(v.X, v.Y), onScreen
    end

    local function GetParts(char)
        if not char then return nil end
        return {
            Head = char:FindFirstChild("Head"),
            Root = char:FindFirstChild("HumanoidRootPart"),
            Humanoid = char:FindFirstChildOfClass("Humanoid")
        }
    end

    RunService.RenderStepped:Connect(function()
        if not ESPSettings.Enabled then return end
        local t = tick()
        local view = Camera.ViewportSize
        local topY = 1

        for player, esp in pairs(ESPObjects) do
            local char = player.Character
            if not char then
                HideAll(esp)
            else
                local p = GetParts(char)
                if not p or not p.Head or not p.Root or not p.Humanoid then
                    HideAll(esp)
                else
                    local head2D, headVis = ToScreen(p.Head.Position + Vector3.new(0, 0.3, 0))
                    local root2D, rootVis = ToScreen(p.Root.Position)
                    if not rootVis then
                        if esp.Line then esp.Line.Visible = false end
                        if esp.Box then esp.Box.Visible = false end
                        if esp.Name then esp.Name.Visible = false end
                        if esp.HealthFill then esp.HealthFill.Visible = false end
                        if esp.HealthOutline then esp.HealthOutline.Visible = false end
                    else
                        -- BOX
                        if esp.Box then
                            local minX, minY = math.huge, math.huge
                            local maxX, maxY = -math.huge, -math.huge
                            local any = false
                            for _, part in ipairs(char:GetDescendants()) do
                                if part:IsA("BasePart") then
                                    local sPos, sVis = ToScreen(part.Position)
                                    if sVis then
                                        any = true
                                        minX = math.min(minX, sPos.X)
                                        maxX = math.max(maxX, sPos.X)
                                        minY = math.min(minY, sPos.Y)
                                        maxY = math.max(maxY, sPos.Y)
                                    end
                                end
                            end
                            if any then
                                esp.Box.Visible = ESPSettings.Box
                                esp.Box.Position = Vector2.new(minX, minY)
                                esp.Box.Size = Vector2.new(math.max(1, maxX - minX), math.max(1, maxY - minY))
                                esp.Box.Color = RainbowColor(t)
                            else
                                esp.Box.Visible = false
                            end
                        end

                        -- LINHA vindo do topo
                        if esp.Line then
                            esp.Line.Visible = ESPSettings.Line
                            esp.Line.From = Vector2.new(view.X / 2, topY)
                            esp.Line.To = root2D
                            esp.Line.Color = RainbowColor(t)
                        end

                        -- NOME
                        if esp.Name then
                            esp.Name.Visible = ESPSettings.Name and headVis
                            if headVis then
                                esp.Name.Position = Vector2.new(head2D.X, head2D.Y - 15)
                                esp.Name.Text = player.Name
                            end
                        end

                        -- VIDA
                        if esp.HealthFill and esp.HealthOutline then
                            local boxVisible = esp.Box and esp.Box.Visible
                            if boxVisible and p.Humanoid then
                                local ratio = clamp(p.Humanoid.Health / math.max(1, p.Humanoid.MaxHealth), 0, 1)
                                local boxPos = esp.Box.Position
                                local boxSize = esp.Box.Size
                                local x = boxPos.X + boxSize.X + 6
                                local yBottom = boxPos.Y + boxSize.Y
                                local yTop = boxPos.Y

                                esp.HealthOutline.Visible = true
                                esp.HealthOutline.From = Vector2.new(x, yBottom)
                                esp.HealthOutline.To = Vector2.new(x, yTop)

                                esp.HealthFill.Visible = true
                                esp.HealthFill.From = Vector2.new(x, yBottom)
                                esp.HealthFill.To = Vector2.new(x, yBottom - (boxSize.Y * ratio))

                                local r = math.floor((1 - ratio) * 255)
                                local g = math.floor(ratio * 255)
                                esp.HealthFill.Color = Color3.fromRGB(r, g, 0)
                            else
                                esp.HealthFill.Visible = false
                                esp.HealthOutline.Visible = false
                            end
                        end
                    end
                end
            end
        end
    end)
end)
