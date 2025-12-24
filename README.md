-- SCRIPT CENTRAL DE MISSÃO E STEALTH
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

-- Configurações
local VIEW_DISTANCE = 45
local VIEW_ANGLE = 50
local GameEvent = ReplicatedStorage:WaitForChild("GameEvent")

----------------------------------------------------------------
-- 1. LÓGICA DO SERVIDOR (DETECÇÃO E MISSÃO)
----------------------------------------------------------------

local function checkVision()
	for _, player in pairs(Players:GetPlayers()) do
		local char = player.Character
		if not char or not char:FindFirstChild("HumanoidRootPart") then continue end
		
		local playerPos = char.HumanoidRootPart.Position
		
		for _, npc in pairs(workspace.NPCs:GetChildren()) do
			local head = npc:FindFirstChild("Head")
			if head then
				local dirToPlayer = (playerPos - head.Position).Unit
				local angle = math.deg(math.acos(head.CFrame.LookVector:Dot(dirToPlayer)))
				local dist = (playerPos - head.Position).Magnitude
				
				if angle < VIEW_ANGLE and dist < VIEW_DISTANCE then
					-- Raycast para obstrução (Paredes)
					local rayParams = RaycastParams.new()
					rayParams.FilterDescendantsInstances = {npc, char}
					rayParams.FilterType = Enum.RaycastFilterType.Exclude
					
					local result = workspace:Raycast(head.Position, dirToPlayer * VIEW_DISTANCE, rayParams)
					
					if not result then -- Caminho limpo até o jogador
						GameEvent:FireClient(player, "Detected", npc.Name)
					end
				end
			end
		end
	end
end

-- Loop de verificação do servidor
task.spawn(function()
	while true do
		checkVision()
		task.wait(0.3)
	end
end)

----------------------------------------------------------------
-- 2. LÓGICA DO CLIENTE (UI E ESP) - Gerada via Script
----------------------------------------------------------------

-- Este trecho cria um LocalScript automaticamente para cada jogador que entrar
Players.PlayerAdded:Connect(function(player)
	player.CharacterAdded:Connect(function(char)
		local localScript = Instance.new("LocalScript")
		localScript.Name = "ClientStealthHandler"
		
		localScript.Source = [[
			local player = game.Players.LocalPlayer
			local event = game.ReplicatedStorage:WaitForChild("GameEvent")
			local RunService = game:GetService("RunService")
			
			-- Criar Interface Simples
			local gui = Instance.new("ScreenGui", player.PlayerGui)
			local label = Instance.new("TextLabel", gui)
			label.Size = UDim2.new(0, 200, 0, 50)
			label.Position = UDim2.new(0.5, -100, 0, 50)
			label.Text = "MISSÃO: Infiltre-se e Roube o Item"
			label.BackgroundColor3 = Color3.new(0,0,0)
			label.TextColor3 = Color3.new(1,1,1)

			-- Sistema de ESP (Destaque Visual)
			local function applyESP()
				for _, npc in pairs(workspace.NPCs:GetChildren()) do
					if not npc:FindFirstChild("Highlight") then
						local h = Instance.new("Highlight", npc)
						h.FillColor = Color3.fromRGB(255, 0, 0)
						h.FillTransparency = 0.6
					end
				end
				for _, obj in pairs(workspace.Objectives:GetChildren()) do
					if not obj:FindFirstChild("Highlight") then
						local h = Instance.new("Highlight", obj)
						h.FillColor = Color3.fromRGB(255, 215, 0)
					end
				end
			end

			-- Alerta de Detecção
			event.OnClientEvent:Connect(function(mode, npcName)
				if mode == "Detected" then
					label.Text = "DETECTADO POR: " .. npcName
					label.BackgroundColor3 = Color3.new(1, 0, 0)
					task.wait(2)
					label.BackgroundColor3 = Color3.new(0, 0, 0)
					label.Text = "MISSÃO: Fuja ou Esconda-se!"
				end
			end)

			RunService.RenderStepped:Connect(applyESP)
		]]
		localScript.Parent = player.PlayerGui
	end)
end)

print("Sistema de Stealth Autoral Carregado com Sucesso!")
