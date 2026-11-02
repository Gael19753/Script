local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
 
local player = Players.LocalPlayer
local char = script.Parent
local hum = char:WaitForChild("Humanoid")
local playing = workspace:WaitForChild("Playing", 10)
local animate = char:WaitForChild("Animate", 5)
 
local ID_IMAGEM = "rbxassetid://16861351286"
 
local DURACAO = 5            -- segundos correndo até zerar a estamina
local RECARGA = 8            -- segundos pra recarregar tudo
local ESPERA_ZERO = 2        -- segundos fechada depois de zerar
local MIN_PARA_CORRER = 0    -- 0 = dá pra correr com qualquer estamina
local LARGURA = 0.35         -- largura da barra (parte da tela)
local ALTURA = 12            -- espessura da barra
local TEMPO_ANIM = 0.3       -- velocidade de abrir/fechar
local VEL_NORMAL = hum.WalkSpeed
 
local running = Enum.HumanoidStateType.Running
local runningNoPhysics = Enum.HumanoidStateType.RunningNoPhysics
 
-- interface
local gui = Instance.new("ScreenGui")
gui.Name = "CorridaGui"
gui.ResetOnSpawn = true
gui.DisplayOrder = 10
gui.Parent = player:WaitForChild("PlayerGui")
 
-- botão com a sua imagem (fixa, não muda ao clicar)
local botao = Instance.new("ImageButton")
botao.Size = UDim2.new(0, 110, 0, 110)
botao.Position = UDim2.new(1, -250, 1, -240)
botao.BackgroundTransparency = 1
botao.BorderSizePixel = 0
botao.AutoButtonColor = false
botao.Image = ID_IMAGEM
botao.ScaleType = Enum.ScaleType.Fit
botao.Parent = gui
 
-- barra de estamina (retangular, fundo preto, preenchimento branco)
local fundo = Instance.new("Frame")
fundo.AnchorPoint = Vector2.new(0.5, 0.5)
fundo.Size = UDim2.new(0, 0, 0, ALTURA)
fundo.Position = UDim2.new(0.5, 0, 0.88, 0)
fundo.BackgroundColor3 = Color3.new(0, 0, 0)
fundo.BorderSizePixel = 0
fundo.Visible = false
fundo.Parent = gui
 
local barra = Instance.new("Frame")
barra.Size = UDim2.new(1, 0, 1, 0)
barra.BackgroundColor3 = Color3.new(1, 1, 1)
barra.BorderSizePixel = 0
barra.Parent = fundo
 
if not playing then
    botao.ImageColor3 = Color3.fromRGB(120, 120, 120)
    warn("BoolValue Playing não encontrado no Workspace")
    return
end
 
-- animação de abrir/fechar (as pontas vão até o meio)
local infoAnim = TweenInfo.new(TEMPO_ANIM, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local barraAberta = false
local tweenBarra
 
local function abrirBarra()
    barraAberta = true
    fundo.Visible = true
    if tweenBarra then tweenBarra:Cancel() end
    tweenBarra = TweenService:Create(fundo, infoAnim, {Size = UDim2.new(LARGURA, 0, 0, ALTURA)})
    tweenBarra:Play()
end
 
local function fecharBarra()
    barraAberta = false
    if tweenBarra then tweenBarra:Cancel() end
    local t = TweenService:Create(fundo, infoAnim, {Size = UDim2.new(0, 0, 0, ALTURA)})
    tweenBarra = t
    t.Completed:Connect(function(estado)
        if estado == Enum.PlaybackState.Completed and not barraAberta then
            fundo.Visible = false
        end
    end)
    t:Play()
end
 
-- lógica
local energia = 1
local ativo = false
local bloqueado = false
local antes = false
local resetando = false
 
local function parar()
    if not ativo then return end
    ativo = false
    playing.Value = false
    hum.WalkSpeed = VEL_NORMAL
end
 
local function comecar()
    if ativo or bloqueado or energia <= MIN_PARA_CORRER or hum.Health <= 0 then return end
    ativo = true
end
 
-- volta a animação padrão do Roblox (sem travar nem deslizar)
local function resetarAnimacao()
    if not animate or resetando then return end
    resetando = true
    task.spawn(function()
        animate.Disabled = true
        task.wait()
        animate.Disabled = false
        task.wait(0.1)
        -- empurrãozinho na velocidade pra o Animate perceber que está andando
        if not ativo then
            hum.WalkSpeed = VEL_NORMAL + 2
            task.wait(0.1)
            if not ativo then
                hum.WalkSpeed = VEL_NORMAL
            end
        end
        resetando = false
    end)
end
 
botao.Activated:Connect(function()
    if ativo then
        parar()
    else
        comecar()
    end
end)
 
RunService.RenderStepped:Connect(function(dt)
    local estado = hum:GetState()
    local andando = (estado == running or estado == runningNoPhysics)
    and hum.MoveDirection.Magnitude > 0
    
    -- a animação só toca com o botão ativo E o personagem andando
    playing.Value = ativo and andando
    
    if bloqueado then
        -- zerada: espera fechada, sem recarregar
    elseif ativo and andando then
        energia -= dt / DURACAO
        if energia <= 0 then
            energia = 0
            parar()
            bloqueado = true
            task.delay(ESPERA_ZERO, function()
                bloqueado = false
            end)
        end
    else
        -- parado ou sem correr: recarrega
        energia = math.min(1, energia + dt / RECARGA)
    end
    
    barra.Size = UDim2.new(energia, 0, 1, 0)
    
    -- a barra aparece se estiver correndo ou recarregando; some quando cheia ou zerada
    local desejada = (ativo or energia < 1) and not bloqueado
    if desejada and not barraAberta then
        abrirBarra()
    elseif not desejada and barraAberta then
        fecharBarra()
    end
    
    -- no instante em que a corrida para, volta a animação normal
    if antes and not playing.Value then
        resetarAnimacao()
    end
    antes = playing.Value
end)
 
hum.Died:Connect(parar)
