# CS_Monochrome
3D Game Monochrome

<p align="center">
    <img src="https://pbs.twimg.com/media/Bl3kGWqCIAAjnam?format=jpg&name=large" width="900">
</p>

Site web du jeu: http://monochrome.antarka.com/

## Requirements
- [CraftStudio](https://sparklinlabs.itch.io/craftstudio)

## Getting Started
Clone the repository at: **AppData\Roaming\CraftStudio\CraftStudioServer\Projects**.
```
$ git clone https://github.com/AntarkaGame/CS_Monochrome.git 651daf8d-118b-476a-b092-2fdd2350ef6e
```

> ⚠️ Please be sure to keep the uuid.

## Videos

- Release 1: https://www.youtube.com/watch?v=r8lOVQx860A
- Release 1.1: https://www.youtube.com/watch?v=VFK_N2aq7Fo
- Release 1.2: https://www.youtube.com/watch?v=rxTLNU0JsAY

## Soundtracks (by Irvin Montes)

- https://www.youtube.com/watch?v=R7ypSKKw1l4
- https://www.youtube.com/watch?v=sZt29yMZq8I
- https://www.youtube.com/watch?v=M2R8s9VAUSI
- https://www.youtube.com/watch?v=2mrQtXvinxw
- https://www.youtube.com/watch?v=j2WILAwwpNw

## Team

- [Fraxken](https://github.com/fraxken) - GENTILHOMME Thomas &lt;gentilhomme.thomas@gmail.com&gt;
- [Aleksander](https://github.com/AlexandreMalaj) - MALAJ Alexandre &lt;alexandre.malaj@gmail.com&gt;
- [Chlorodatafile](https://github.com/chlorodatafile) - CARON Gwenaël &lt;gwenael.caron@outlook.com&gt;
- [Captainfive](https://github.com/Captainfive) - MONTES Irvin &lt;irvin_montes@outlook.fr&gt;

## Some models

| The robot |
| --- | 
| <img src="https://monochrome.antarka.com/images/robot.png" height="200"> |

## Memory code

<details><summary>Player movement behavior</summary>

```lua
-- ALL RIGHT RESERVED ANTARKA TEAM --
-- Script patch 0.7.4 Alpha
-- Développeur en charge : Fraxken, Shqiptar, Chlorodatafile

-- Déclaration des variables globales
Sprint_statuts = false

function Behavior:Awake()

    CS.Physics.SetGravity( Vector3:New( 0 ,-100, 0 ) )
    
    -- > Jump
    self.nbJump = 0
    self.nbJumpMax = 3
    self.Jumping = false
    self.Jumping_statuts = true
    self.RunningStatuts = false
    self.DoubleJumping = false
    
    -- > Jump velocity
    self.Velocity_Jump = 39
    self.Velocity_Jump_Sprint = 45
    self.Velocity_WallJump = 35.5
    self.Velocity_WallJump_Sprint = 47.5
    
    -- > Sprint
    self.RunningStatuts = false
    self.Speed_activate = false
    self.Speed_basic = self.Speed
    self.Speed_max = self.Speed_basic + self.Speed_add
    self.Speed_decrease = 6
    self.Speed_increase = 4 
    self.Speed_StraffOut = 1.35 -- Anti straff
    
    -- > Stamina
    self.Max_stamina = 240
    self.Stamina = 240
    self.Stamina_conso = 0.25
    self.Stamina_conso_nb = 15
    self.Stamina_regen_nb = 15
    self.Stamina_regen_delay = 0.175
    self.Stamina_regen_time = 2.1
    self.Stamina_run = nil 
    self.Stamina_run_regen = nil
    self.Stamina_run_delay = nil
    self.Stamina_anime_frame = 1000
    self.Stamina_JumpSprint_cost = 10
    self.Stamina_Jump_cost = 0
    self.Stamina_Jump_cost_division = 2 
    
    --> Stamina object
    self.Stamina_object = CS.FindGameObject("Stamina")
    self.Stamina_model  = CS.FindGameObject("Stamina"):GetComponent("ModelRenderer")
    self.Stamina_texte  = CS.FindGameObject("Stamina_texte"):GetComponent("TextRenderer")
    
    --> Define defaut stamina value
    self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
    self.Stamina_model:SetAnimation( Anim.Hud.Stamina ) 
    self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
    self.Stamina_model:StopAnimationPlayback()
    
    --> Sound
    self.sound_saut = CS.FindAsset( "Sound/saut" )
    self.Instance_saut = self.sound_saut:CreateInstance()
    self.Instance_saut:SetLoop( false )
    self.Instance_saut:SetVolume( 0.5 )
    
    self.sound_wall_saut = CS.FindAsset( "Sound/wallJump" )
    self.Instance_wall_saut = self.sound_wall_saut:CreateInstance()
    self.Instance_wall_saut:SetLoop( false )
    self.Instance_wall_saut:SetVolume( 0.5 )
    
    self.sound_walk = CS.FindAsset( "Sound/walk" )
    self.Instance_walk = self.sound_walk:CreateInstance()
    self.Instance_walk:SetLoop( true )
    self.Instance_walk:SetVolume( 0.5 )
    
    self.sound_sprint = CS.FindAsset( "Sound/Sprint" )
    self.Instance_sprint = self.sound_sprint:CreateInstance()
    self.Instance_sprint:SetLoop( true )
    self.Instance_sprint:SetVolume( 0.5 )
    
    self.sound_presprint = CS.FindAsset( "Sound/PreSprint" )
    self.Instance_presprint = self.sound_presprint:CreateInstance()
    self.Instance_presprint:SetLoop( false )
    self.Instance_presprint:SetVolume( 1.0 )
    
    -- > Friction only Y axes
    self.gameObject.physics:SetFriction(0)
    
    -- > Pick map
    self.map = CS.FindGameObject("Map"):GetComponent("MapRenderer")
    
    self.err_nb = 0
    
    -- > Checkpoint
    self.Checkpoint = nil
    self.Hud_container = CS.FindGameObject("Camera_hud")
    
end

function Behavior:Data(data) 
    if data.Message == "Checkpoint" then
        self.Checkpoint = data.Value
    end
end

function Behavior:Update()
    
    -- > Si la souris est bloquer 
    if LockMouse == true then

        self.gameObject.physics:SetFriction(0)
        
        --[[ > CheckPoint Système 
        if CS.Input.WasButtonJustPressed("R") and self.Checkpoint ~= nil then
            self.Checkpoint_obj = CS.FindGameObject(self.Checkpoint) 
            self.Checkpoint_obj_pos = self.Checkpoint_obj.transform:GetPosition()
            self.gameObject.physics:WarpPosition( Vector3:New( self.Checkpoint_obj_pos.x, self.Checkpoint_obj_pos.y + 4 , self.Checkpoint_obj_pos.z  ) ) 
            if self.Checkpoint == "Start_checkpoint" then
                self.Hud_container:SendMessage("Data",{Message="time",Value=0})
                self.Stamina = self.Max_stamina
                self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                self.RunningStatuts = false
            end
        end --]]
        
        if CS.Input.WasButtonJustPressed("LEFTSHIFT") and CS.Input.IsButtonDown("Z") and self.RunningStatuts == false and self.Instance_presprint:GetState() == SoundInstance.State.Stopped then
            self.Instance_presprint:Play()
        end
        
        -- > Si le joueur appuie sur Sprint (LeftShift)
        if CS.Input.IsButtonDown("LEFTSHIFT") and CS.Input.IsButtonDown("Z") and self.RunningStatuts == false then
        
            --> Sprint sound Check on 
            if self.Instance_sprint:GetState() == SoundInstance.State.Stopped then
                self.Instance_sprint:Play()
            end
            
            -- > Vérification des delay inverse
            if self.Stamina_run_regen ~= nil then
                self.Stamina_run_regen = nil
                self.Stamina_run_delay = nil
            end
            
            -- > Gestion de la stamina
            if (self.Stamina - self.Stamina_regen_nb) > 0 and self.Stamina_run == nil then
                self.Stamina_run = 0 
            elseif (self.Stamina - self.Stamina_regen_nb) <= 0 then
                self.Stamina = 0 
                self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                self.RunningStatuts = true
            else
                if self.Stamina_run < self.Stamina_conso then
                    self.Stamina_run = self.Stamina_run + 1 / 60 
                else
                    self.Stamina_run = nil
                    self.Stamina = self.Stamina - self.Stamina_conso_nb
                    self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                    self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                    self.Speed_activate = true
                end
            end
            
            -- > Incrémentation de la vitesse
            if self.Speed_activate == true then
                if (self.Speed_max - self.Speed) >= 0 then
                    self.Speed = self.Speed + self.Speed_increase
                end
                self.Speed_activate = false
            end
            
            --> Put true sprint
            Sprint_statuts = true 
        else
        
            --> Sprint sound Check off
            if self.Instance_sprint:GetState() == SoundInstance.State.Playing then
                self.Instance_sprint:Stop()
            end
            
            --> Si RunningStatuts == true et que Sprint n'est pas appuiyez.
            if CS.Input.IsButtonDown("LEFTSHIFT") == false and self.Stamina_run_regen == nil then
                self.Stamina_run_regen = 0
            end
            
            --> Gestion stamina
            if self.Stamina_run_regen ~= nil then
                if self.Stamina_run_regen >= self.Stamina_regen_time and self.Stamina_run_delay == nil then
                    self.Stamina_run_delay = 0
                else
                    self.Stamina_run_regen = self.Stamina_run_regen + 1 / 60 
                end
            end
            
            --> Stamina regen
            if self.Stamina_run_delay ~= nil then
                if self.Stamina_run_delay >= self.Stamina_regen_delay then
                    if (self.Stamina + self.Stamina_regen_nb) >= self.Max_stamina then
                        self.Stamina = self.Max_stamina
                        self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                        self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                    else
                        self.Stamina = self.Stamina + self.Stamina_regen_nb
                        self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                        self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                        self.RunningStatuts = false
                    end
                    self.Stamina_run_delay = 0
                else
                    self.Stamina_run_delay = self.Stamina_run_delay + 1 / 60
                end
            end 
            
            --> Vérification de la vitesse
            if self.Speed > self.Speed_basic then
                self.Speed = self.Speed - self.Speed_decrease / 60
            else
                if self.Speed ~= self.Speed_basic then
                    self.Speed = self.Speed_basic 
                end 
            end
            
            --> Put false sprint
            Sprint_statuts = false
        end
        
        
        -- > Execution du son walk 
        if CS.Input.IsButtonDown("Z") or CS.Input.IsButtonDown("S") or CS.Input.IsButtonDown("D") or CS.Input.IsButtonDown("Q") and self.Jumping == false then
            if Sprint_statuts == false then
                if self.Instance_walk:GetState() == SoundInstance.State.Stopped then
                    self.Instance_walk:Play()
                end
            else
                if self.Instance_walk:GetState() == SoundInstance.State.Playing then
                    self.Instance_walk:Stop()
                end
            end
        else
            if self.Instance_walk:GetState() == SoundInstance.State.Playing then
                self.Instance_walk:Stop()
            end
        end
        
        self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y) 
        
        -- > Mouvement script
        if CS.Input.IsButtonDown("Z") and CS.Input.IsButtonDown("S") or CS.Input.IsButtonDown("UP") and CS.Input.IsButtonDown("DOWN") then
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.gameObject.physics:SetLinearVelocity(Vector3:New(0,self.Velocity,0))
        elseif CS.Input.IsButtonDown("Q") and CS.Input.IsButtonDown("D") or CS.Input.IsButtonDown("LEFT") and CS.Input.IsButtonDown("RIGHT") then
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.gameObject.physics:SetLinearVelocity(Vector3:New(0,self.Velocity,0))
        elseif CS.Input.IsButtonDown("Z") and CS.Input.IsButtonDown("D") or CS.Input.IsButtonDown("UP") and CS.Input.IsButtonDown("RIGHT") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y) 
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New( self.Speed * - math.sin(self.OrientPlayer - math.pi / 4), self.Velocity, self.Speed * - math.cos(self.OrientPlayer - math.pi / 4) )
            self.gameObject.physics:SetLinearVelocity(self.NewPos)
        elseif CS.Input.IsButtonDown("Z") and CS.Input.IsButtonDown("Q") or CS.Input.IsButtonDown("UP") and CS.Input.IsButtonDown("LEFT") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y)
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New(self.Speed * - math.sin(self.OrientPlayer + math.pi / 4),self.Velocity,self.Speed * - math.cos(self.OrientPlayer + math.pi / 4))
            self.gameObject.physics:SetLinearVelocity(self.NewPos)
        elseif CS.Input.IsButtonDown("S") and CS.Input.IsButtonDown("D") or CS.Input.IsButtonDown("DOWN") and CS.Input.IsButtonDown("RIGHT") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y)
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New(self.Speed * math.sin(self.OrientPlayer + math.pi / 4),self.Velocity,self.Speed * math.cos(self.OrientPlayer + math.pi / 4))
            self.gameObject.physics:SetLinearVelocity(self.NewPos)
        elseif CS.Input.IsButtonDown("S") and CS.Input.IsButtonDown("Q") or CS.Input.IsButtonDown("DOWN") and CS.Input.IsButtonDown("LEFT") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y)
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New(self.Speed * math.sin(self.OrientPlayer - math.pi / 4),self.Velocity,self.Speed * math.cos(self.OrientPlayer - math.pi / 4))
            self.gameObject.physics:SetLinearVelocity(self.NewPos)  
        elseif CS.Input.IsButtonDown("Z") or CS.Input.IsButtonDown("UP") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y) 
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New(self.Speed * - math.sin(self.OrientPlayer),self.Velocity,self.Speed * - math.cos(self.OrientPlayer))
            self.gameObject.physics:SetLinearVelocity(self.NewPos)  
        elseif CS.Input.IsButtonDown("Q") or CS.Input.IsButtonDown("LEFT") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y)
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New((self.Speed / self.Speed_StraffOut) * - math.sin(self.OrientPlayer + math.pi / 2),self.Velocity,(self.Speed / self.Speed_StraffOut) * - math.cos(self.OrientPlayer + math.pi / 2))
            self.gameObject.physics:SetLinearVelocity(self.NewPos)
        elseif CS.Input.IsButtonDown("S") or CS.Input.IsButtonDown("DOWN") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y) 
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New((self.Speed / self.Speed_StraffOut) * math.sin(self.OrientPlayer),self.Velocity,(self.Speed / self.Speed_StraffOut) * math.cos(self.OrientPlayer))
            self.gameObject.physics:SetLinearVelocity(self.NewPos)
        elseif CS.Input.IsButtonDown("D") or CS.Input.IsButtonDown("RIGHT") then
            self.OrientPlayer = math.rad( Cam.transform:GetEulerAngles().y)
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.NewPos = Vector3:New((self.Speed / self.Speed_StraffOut) * - math.sin(self.OrientPlayer - math.pi / 2),self.Velocity,(self.Speed / self.Speed_StraffOut) * - math.cos(self.OrientPlayer - math.pi / 2))
            self.gameObject.physics:SetLinearVelocity(self.NewPos)
        else
            self.Velocity = self.gameObject.physics:GetLinearVelocity().y
            self.gameObject.physics:SetLinearVelocity(Vector3:New(0,self.Velocity,0))
        end
        
        -- > Ray qui part vers le bas
        self.RayJumpDown        = Ray:New(self.gameObject.transform:GetPosition(),Vector3:New(0,-1,0))
        self.RayJumpDistDown    = self.RayJumpDown:IntersectsMapRenderer( self.map )
            
        -- > Jump normal
        if self.RayJumpDistDown <= self.MaxJump then
        
            
            -- > Sauter est appuiyez
            if CS.Input.WasButtonJustPressed("ESPACE")  then
                self.Velociter = self.gameObject.physics:GetLinearVelocity()
                
                -- > Vérification du sprint ou non
                if Sprint_statuts == true then
                    self.gameObject.physics:SetLinearVelocity(Vector3:New(self.Velociter.x, self.Velocity_Jump_Sprint , self.Velociter.z))
                    self.Stamina = self.Stamina - (self.Stamina_JumpSprint_cost / self.Stamina_Jump_cost_division)
                    self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                    self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                else
                    self.gameObject.physics:SetLinearVelocity(Vector3:New(self.Velociter.x, self.Velocity_Jump , self.Velociter.z))
                    self.Stamina = self.Stamina - (self.Stamina_Jump_cost / self.Stamina_Jump_cost_division)
                    self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                    self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                end
                
                -- > Instance de son 
                if self.Instance_saut:GetState() ~= SoundInstance.State.Playing then
                    self.Instance_saut:Play()
                end
                
                self.Jumping = true
                self.nbJump = 1
            else
                self.Jumping = false
                self.nbJump = 0
            end
            
        else
        
            -- > Instance de son
            if self.Instance_walk:GetState() == SoundInstance.State.Playing then
                self.Instance_walk:Stop()
            end
            
            if self.Instance_sprint:GetState() == SoundInstance.State.Playing then
                self.Instance_sprint:Stop()
            end
            
            if self.nbJump <= self.nbJumpMax and self.nbJump >= 1 then
                if self.RayJumpDistDown <= self.MaxJump then
                    self.nbJump = 0
                end
            elseif self.nbJump == 0 then
                self.Jumping = true
                self.nbJump = 1
            end
            
        end
        
        -- > Wall jump
        if self.nbJump >= 1 and self.nbJump < self.nbJumpMax and self.Jumping == true then
        
            self.RayJumpRight       = Ray:New( self.gameObject.transform:GetPosition(), Vector3.Rotate( Vector3:New(1,0,0),  self.gameObject.transform:GetOrientation() ))
            self.RayJumpleft        = Ray:New( self.gameObject.transform:GetPosition(), Vector3.Rotate( Vector3:New(-1,0,0), self.gameObject.transform:GetOrientation() ))
            self.RayJumpFront       = Ray:New( self.gameObject.transform:GetPosition(), Vector3.Rotate( Vector3:New(0,0,-1), self.gameObject.transform:GetOrientation() ))
            
            self.RayJumpDistRight   = self.RayJumpRight:IntersectsMapRenderer( self.map )
            self.RayJumpDistLeft    = self.RayJumpleft:IntersectsMapRenderer(  self.map )
            self.RayJumpDistFront   = self.RayJumpFront:IntersectsMapRenderer( self.map )
            
            -- > WHY RAY RETURN NIL ???? 
            if self.RayJumpDistRight ~= nil and self.RayJumpDistLeft ~= nil and self.RayJumpDistFront ~= nil then
            
                if self.RayJumpDistRight <= 2 or self.RayJumpDistLeft <= 2 or self.RayJumpDistFront <= 4 then
                    if CS.Input.WasButtonJustPressed("ESPACE") then
                        self.Velociter = self.gameObject.physics:GetLinearVelocity()
                        if Sprint_statuts == true then
                            self.gameObject.physics:SetLinearVelocity(Vector3:New(self.Velociter.x, self.Velocity_WallJump_Sprint , self.Velociter.z))
                            self.Stamina = self.Stamina - self.Stamina_JumpSprint_cost
                            self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                            self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                        else
                            self.gameObject.physics:SetLinearVelocity(Vector3:New(self.Velociter.x, self.Velocity_WallJump , self.Velociter.z))
                            self.Stamina = self.Stamina - self.Stamina_Jump_cost
                            self.Stamina_texte:SetText(tostring(self.Stamina).." / "..tostring(self.Max_stamina))
                            self.Stamina_model:SetAnimationTime( (self.Stamina_anime_frame / self.Max_stamina * self.Stamina ) / 30 )
                        end
                        
                        if self.Instance_wall_saut:GetState() == SoundInstance.State.Stopped then
                            self.Instance_wall_saut:Play()
                        end
                        
                        self.nbJump = self.nbJump + 1
                    end
                end
            
            else
                --print("Erreur "..self.err_nb.." :NIL: survenue")
                --self.err_nb = self.err_nb + 1 
            end

            
        end
        
    else
        self.gameObject.physics:SetFriction(1)
    end
    
end
```

</details>

<details><summary>Bumper by Aleksander (with our first cubic interpolation)</summary>


```lua
-- ALL RIGHT RESERVED ANTARKA TEAM --
-- Script patch 0.1 Alpha
-- Développeur en charge : Shqiptar

function Behavior:Awake()
    self.a = self.gameObject:FindChild("a")
    self.b = self.gameObject:FindChild("b")
    self.c = self.gameObject:FindChild("c")
    self.d = self.gameObject:FindChild("d")
    
    self.Player = CS.FindGameObject("Player_physics")
    self.timer = nil
end

function Behavior:Update()

    if self.timer ~= nil then
        
        -- > Calcule de la courbe
        self.factor = self.timer / self.duration
        self.out = CubicInterpolate(self.a.transform:GetPosition(), self.b.transform:GetPosition(), self.c.transform:GetPosition(), self.d.transform:GetPosition(), self.factor)

        -- > On warp le joueur
        self.Player.physics:WarpPosition(self.out)
        
        -- > Optionnel
        --self.Inter = CS.CreateGameObject("Inter")
        --self.Inter:CreateComponent("ModelRenderer"):SetModel( CS.FindAsset("Point") )
        --self.Inter.transform:SetPosition(self.out)
        
        self.timer = self.timer + 1 / 60
            
        if self.timer >= self.duration then
            -- > Fin de courbe
            self.Player.physics:SetLinearVelocity(self.Player.physics:GetLinearVelocity())
            self.timer = nil
        end
        
    elseif self.timer == nil and Vector3.Distance(self.Player.transform:GetPosition(),self.gameObject.transform:GetPosition()) <= 5 then
        self.timer = 0
    end
    
end

-- > Function CubicInterpolate
function CubicInterpolate(a, b, c, d , mu)
    local out = Vector3:New(0)
    local a0,a1,a2,a3,mu2
    
    mu2 =mu*mu
    a0 = d.x - c.x - a.x + b.x
    a1 = a.x - b.x - a0
    a2 = c.x - a.x
    a3 = b.x
    out.x = a0*mu*mu2+a1*mu2+a2*mu+a3
    
    a0 = d.y - c.y - a.y + b.y
    a1 = a.y - b.y - a0
    a2 = c.y - a.y
    a3 = b.y
    out.y = a0*mu*mu2+a1*mu2+a2*mu+a3
    
    a0 = d.z - c.z - a.z + b.z
    a1 = a.z - b.z - a0
    a2 = c.z - a.z
    a3 = b.z
    out.z = a0*mu*mu2+a1*mu2+a2*mu+a3
    
    return out
end
```

</details>

<details><summary>2D Camera by fraxken</summary>

```lua
UI,UICamera = nil
PixelToUnit, smallSideSize = nil
MousePos,MouseToPixel,MouseInactive = nil, {x=0,y=0}, 0
Hud, Hud_active = nil, "Accueil"
SGX,SGY = nil
MouseIn,MouseMove,MouseDelta = false, false, nil
ScreenCenter = {x=0,y=0}
ResizeTab = {}
Allow,Allow2 = true
State = nil
Popup_call = false
active_sub = nil
Call_exit_popup = nil
ToFocus_Keyboard = false

function Behavior:Awake()
    State = self.state
    
    if State then 
        Hud = {
            ["Accueil"] = {
                ["Chunk"] = false,
                ["Object"] = CS.FindGameObject("CC_Accueil"),
                ["Multi_container"] = true,
                ["Multi_key_active"] = 1,
                ["Multi_key"] = {
                    CS.FindAsset("Hud/Lobby/Solo","Scene"),
                    CS.FindAsset("Hud/Lobby/Multijoueurs","Scene")
                },
                ["Multi_key_model"] = {
                    CS.FindGameObject("Solo"),
                    CS.FindGameObject("Multijoueurs")
                },
                ["Escape"] = "Exit",
                ["Sub_focus"] = CS.FindGameObject("Solo"),
                ["Sub_focus_model"] = CS.FindAsset("Hud/BootScreen/Menu_sub_focus","Model"),
                ["Style"] = {
                    [0] = {
                        name = "Sub_",
                        pos = {x={"left",0.34},y={"top",3.5}}
                    }
                }
            },
            ["Inventaire"] = {
                ["Chunk"] = false,
                ["Object"] = CS.FindGameObject("CC_Inventaire"),
                ["Multi_container"] = false,
                ["Multi_key_active"] = 1,
                ["Multi_key"] = {
                    CS.FindAsset("Hud/Inventaire/Statistiques","Scene"),
                    CS.FindAsset("Hud/Inventaire/Skins","Scene"),
                    CS.FindAsset("Hud/Inventaire/Success","Scene")
                },
                ["Multi_key_model"] = {
                    CS.FindGameObject("Stats"),
                    CS.FindGameObject("Skins"),
                    CS.FindGameObject("Success")
                },
                ["Escape"] = "Accueil",
                ["Sub_focus"] = CS.FindGameObject("Stats"),
                ["Sub_focus_model"] = CS.FindAsset("Hud/BootScreen/Menu_sub_focus","Model"),
                ["Style"] = {
                    [0] = {
                        name = "Sub_",
                        pos = {x={"left",0.34},y={"top",3.5}}
                    }
                }
            },
            ["Option"] = {
                ["Chunk"] = false,
                ["Object"] = CS.FindGameObject("CC_Option"),
                ["Multi_container"] = true,
                ["Multi_key_active"] = 1,
                ["Multi_key"] = {
                    CS.FindAsset("Hud/Option/Keyboard","Scene"),
                    CS.FindAsset("Hud/Option/Mouse","Scene"),
                    CS.FindAsset("Hud/Option/Hud","Scene"),
                    CS.FindAsset("Hud/Option/Sound","Scene")
                },
                ["Multi_key_model"] = {
                    CS.FindGameObject("Keyboard"),
                    CS.FindGameObject("Mouse"),
                    CS.FindGameObject("Hud"),
                    CS.FindGameObject("Sound")
                },
                ["Escape"] = "Accueil",
                ["Sub_focus"] = CS.FindGameObject("Keyboard"),
                ["Sub_focus_model"] = CS.FindAsset("Hud/BootScreen/Menu_sub_focus","Model"),
                ["Style"] = {
                    [0] = {
                        name = "Sub_",
                        pos = {x={"left",0.34},y={"top",3.5}}
                    }
                }
            }
        }
    end
    
    SGX,SGY = CS.Screen.GetSize().x, CS.Screen.GetSize().y
    UICamera = self.gameObject.camera
    UI = UICamera:CreateRay( CS.Input.GetMousePosition() )
    SmallSide()
    Switch(Hud_active) 
    
    Call_exit_popup = function()
        self.quit_popup = Popup:New("quit_popup",{ "Exit","Exit the game ?", "yes", "no"},nil,function()
            self.quit_popup:Destroy()
            SaveData()
            CS.Exit()
        end)
        self.quit_popup:Call()
    end
end

function UI_Pos(item,x,y,key) 
    if key then
        table.insert(ResizeTab,{item,{x[1],x[2]},{y[1],y[2]}})
    end
    local exec = {x=0,y=0}
    local SCX,SCY = ScreenCenter.x,ScreenCenter.y
    local Back = {x=x[1],y=y[1]}
    local BackPos = {x=x[2],y=y[2]}
    
    -- > On définie la position 
    if x[1] == "left" then x[1] = -1; exec.x = x[1] * SCX + x[2] elseif x[1] == "right" then x[1] = 1; x[2] = -x[2]; exec.x = x[1] * SCX + x[2] else exec.x = 0 end  
    if y[1] == "top" then y[1] = 1; y[2] = -y[2]; exec.y = y[1] * SCY + y[2] elseif y[1] == "bottom" then y[1] = -1; exec.y = y[1] * SCY + y[2] else exec.y = 0 end
    
    local item = CS.FindGameObject(tostring(item)).transform
    -- > On execute la mise en place de la nouvelle position
    item:SetLocalPosition( Vector3:New( 
        exec.x, 
        exec.y, 
        item:GetLocalPosition().z 
    ) ) 
    x[1] = Back.x
    y[1] = Back.y
    x[2] = BackPos.x
    y[2] = BackPos.y
end 

-- > Change orthographic scale
function ScreenSize(size,resize)
    local x = UICamera
    x:SetOrthographicScale( CS.Screen.GetSize().y / tonumber(size) )
    if resize then Resize() end
end

-- > Function qui switch les modules
function Switch(item)
    if Allow then 
        if State then 
            if item == "Exit" then
                Call_exit_popup()
            else
                CS.FindGameObject(Menu_focus).modelRenderer:SetModel( CS.FindAsset( "Hud/BootScreen/Menu_top", "Model") ) 
                CS.FindGameObject(item).modelRenderer:SetModel( CS.FindAsset( "Hud/BootScreen/Menu_top_focus", "Model") )
                Menu_focus = item 
                
                local active = Hud_active 
                Hud[active]["Object"].transform:SetLocalPosition( Vector3:New( 200,0, -5 ) ) 
                Hud[active]["Chunk"] = false
                Hud[item]["Object"].transform:SetLocalPosition( Vector3:New( 0,0, -5 ) ) 
                Hud[item]["Chunk"] = true
                Hud[item]["Sub_focus"].modelRenderer:SetModel( Hud[item]["Sub_focus_model"] ) 
                Hud[item]["BackScape"] = active
                Hud_active = item
                
                if active_sub ~= nil then
                    CS.Destroy( active_sub ) 
                end
                if Hud[item]["Multi_container"] then
                    active_sub = CS.Instantiate( "sub_key", Hud[item]["Multi_key"][Hud[item]["Multi_key_active"]] )
                    Allow2 = false
                end
            end
        end
        Resize()
        Allow = false
    end
end

-- > Resize function
function Resize()
    SGX,SGY = CS.Screen.GetSize().x, CS.Screen.GetSize().y
    ScreenSize(16)
    SmallSide()
    ScreenCenter.x,ScreenCenter.y = SGX*PixelToUnit / 2, SGY*PixelToUnit / 2
    if State then 
        if Hud[Hud_active]["Multi_container"] then
            UI_Pos("Sub_Container",{"left",1},{"top",6.5},false)
        end
    end
    for i = 0, #ResizeTab do
        if ResizeTab[i] ~= nil then
            UI_Pos(ResizeTab[i][1],ResizeTab[i][2],ResizeTab[i][3],false)
        end
    end 
    if State then 
        for i = 0, #Hud[Hud_active]["Style"] do
            UI_Pos(tostring(Hud[Hud_active]["Style"][i].name..Hud_active),Hud[Hud_active]["Style"][i].pos.x,Hud[Hud_active]["Style"][i].pos.y,false)
        end
    end
end

-- > Return the smallest screen side
function SmallSide()
    smallSideSize = CS.Screen.GetSize().y
    if CS.Screen.GetSize().x < CS.Screen.GetSize().y then
        smallSideSize = CS.Screen.GetSize().x
    end
    PixelToUnit = UICamera:GetOrthographicScale() / smallSideSize 
end

function Behavior:Update()
    
    Allow = true
    MousePos = CS.Input.GetMousePosition()
    MouseDelta = CS.Input.GetMouseDelta()
    UI = self.gameObject.camera:CreateRay( MousePos )
    
    if not Allow2 and State then
        Allow2 = true
        UI_Pos("Sub_Container",{"left",1},{"top",6.5},false)
    end
    
    -- > On détecte si la souris bouge ou non
    if MouseDelta.x ~= 0 or MouseDelta.y ~= 0 and MouseIn then
        MouseMove = true
        MouseInactive = 0
    else
        MouseMove = false
        MouseInactive = MouseInactive + 1 / 60 
    end
    
    -- > On détecte si la souris est en dehors de la fenêtre
    if MousePos.x >= 0 and MousePos.x <= SGX and MousePos.y >= 0 and MousePos.y <= SGY then
        if MouseIn == false then
            MouseIn = true
        end
        MouseToPixel.x = MousePos.x * PixelToUnit
        MouseToPixel.y = MousePos.y * PixelToUnit
    else
        if MouseIn == true then 
            MouseIn = false
        end
    end
    
    -- > On vérifie si la fenêtre est redimensionner 
    if SGX ~= CS.Screen.GetSize().x or SGY ~= CS.Screen.GetSize().y then
        Resize()
    end
    
end
```

</details>