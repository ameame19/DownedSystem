-- server side

local remote = game.ReplicatedStorage:WaitForChild('RemoteEvent')

local Players = game:GetService("Players")

local function setupDownedMechanic(player)
	player.CharacterAdded:Connect(function(char)
		local hum = char:WaitForChild("Humanoid")
		hum.BreakJointsOnDeath = false 

		hum.HealthChanged:Connect(function(health)
			local isDowned = char:FindFirstChild("IsDowned")

			-- Catch health before 0 to stop "Direct Death"
			if health <= 0 and not isDowned then
				hum.Health = 1
			end

			if isDowned then 
				if health <= 0 then hum.Health = 0.1 end 
				return 
			end

			-- Trigger Downed Logic at 15 HP
			if health <= 2 and health > 0 then
				local tag = Instance.new("BoolValue")
				tag.Name = "IsDowned"
				tag.Parent = char

				hum.Health = 25 
				hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
				hum.WalkSpeed = 4
				hum.PlatformStand = true 
				hum:UnequipTools()

				local prompt = Instance.new("ProximityPrompt")
				prompt.Name = "HelpPrompt"
				prompt.ActionText = "Revive " .. player.Name
				prompt.HoldDuration = 2 
				prompt.Parent = char:WaitForChild("HumanoidRootPart")

				-- 20 Second Death Timer
				task.delay(20, function()
					if char:FindFirstChild("IsDowned") == tag then
						tag:Destroy() 
						if prompt then prompt:Destroy() end

						hum.PlatformStand = false
						hum:SetStateEnabled(Enum.HumanoidStateType.Dead, true)

						task.wait(0.2) -- Safety wait to prevent crash
						hum.Health = 0
						char:BreakJoints() 
					end
				end)

				prompt.Triggered:Connect(function(helper)
					if helper.UserId == player.UserId then return end -- No self-revive
					tag:Destroy() 
					if prompt then prompt:Destroy() end

					hum.PlatformStand = false
					hum:SetStateEnabled(Enum.HumanoidStateType.Dead, true)
					hum.Health = 70 
					hum.WalkSpeed = 16
					hum:ChangeState(Enum.HumanoidStateType.GettingUp)
				end)
			end
		end)
	end)
end

Players.PlayerAdded:Connect(setupDownedMechanic)

remote.OnServerEvent:Connect(function(plr)
	plr:LoadCharacter()
end)

-- local script
local Players = game:GetService("Players")
local StarterGui = game:GetService("StarterGui")
local LocalPlayer = Players.LocalPlayer
local remote = game.ReplicatedStorage:WaitForChild('RemoteEvent')

local Players = game:GetService("Players")
local StarterGui = game:GetService("StarterGui")
local LocalPlayer = Players.LocalPlayer
local remote = game.ReplicatedStorage:WaitForChild('RemoteEvent')

local playerGui = LocalPlayer:WaitForChild("PlayerGui")
local screenGui = playerGui:WaitForChild("ScreenGui")
local textlable = screenGui:WaitForChild("TextLabel")

local char = script.Parent
local hum = char:WaitForChild("Humanoid")
local animator = hum:WaitForChild("Animator")
local animation = script:FindFirstChild("Animation")
local animTrack = animator:LoadAnimation(animation)

textlable.Visible = false

char.ChildAdded:Connect(function(child)
	if child.Name == "IsDowned" then
		-- 1. Setup Animation
		animTrack.Priority = Enum.AnimationPriority.Action
		animTrack.Looped = true
		animTrack:Play()

		-- 2. Setup UI
		StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Backpack, false)
		textlable.Visible = true

		-- 3. FIX FLICKERING: Disable states that fight the animation
		hum:SetStateEnabled(Enum.HumanoidStateType.Running, false)
		hum:SetStateEnabled(Enum.HumanoidStateType.Climbing, false)
		hum:ChangeState(Enum.HumanoidStateType.Physics)

		-- 4. Hide your own revive prompt
		task.spawn(function()
			local hrp = char:WaitForChild("HumanoidRootPart")
			local p = hrp:WaitForChild("HelpPrompt", 5)
			if p then p.Enabled = false end
		end)

		-- 5. Timer & Remote trigger
		task.spawn(function()
			local x = 20
			while x >= 0 and char:FindFirstChild("IsDowned") do
				textlable.Text = "YOU ARE DOWNED! DIE IN: ".. x
				task.wait(1)
				x = x - 1
				if x == 1 then
					remote:FireServer()
				end
			end
		end)
	end
end)

char.ChildRemoved:Connect(function(child)
	if child.Name == "IsDowned" then
		animTrack:Stop(0.1)

		-- Re-enable states so player can move again
		hum:SetStateEnabled(Enum.HumanoidStateType.Running, true)
		hum:SetStateEnabled(Enum.HumanoidStateType.Climbing, true)

		hum:ChangeState(Enum.HumanoidStateType.GettingUp)
		textlable.Visible = false
		StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Backpack, true)
	end
end)
