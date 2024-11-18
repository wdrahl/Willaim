import pygame
import random
import math

pygame.init()
WINDOW_WIDTH = 800
WINDOW_HEIGHT = 600
screen = pygame.display.set_mode((WINDOW_WIDTH, WINDOW_HEIGHT))
pygame.display.set_caption("Sekiro Combat System")
clock = pygame.time.Clock()

# Colors
WHITE = (255, 255, 255)
RED = (255, 0, 0)
BLUE = (0, 0, 255)
BLACK = (0, 0, 0)
YELLOW = (255, 255, 0)

class Player:
    def __init__(self):
        # Combat states
        self.attacking = False
        self.deflecting = False
        
        # Position and movement
        self.x = 400
        self.y = 300
        self.width = 40
        self.height = 40
        self.velocity_x = 0
        self.velocity_y = 0
        self.on_ground = False
        self.jump_speed = -15
        self.move_speed = 8
        self.gravity = 0.8
        self.max_fall_speed = 15
        self.air_control = 0.7
        self.ground_friction = 0.85
        self.air_friction = 0.95
        
        # Combat stats
        self.health = 100
        self.posture = 0
        
        # Dodge mechanics
        self.dodge_speed = 12
        self.dodge_duration = 10
        self.dodge_cooldown = 0
        self.dodge_cooldown_time = 30
        self.is_dodging = False
        self.blocking = False
        self.facing_right = True

        # Hitbox system
        self.attack_direction = "right"
        self.attack_hitbox = pygame.Rect(0, 0, 60, 20)
        self.block_hitbox = pygame.Rect(0, 0, 30, 40)

        # Attack Animation System
        self.attack_frame = 0
        self.attack_animation_timer = 0
        self.attack_animation_frames = 5
        self.attack_frame_duration = 0.067
        self.attack_animation_active = False
        self.attack_sizes = [
            (60, 20),  # Frame 0
            (70, 25),  # Frame 1
            (80, 30),  # Frame 2
            (70, 25),  # Frame 3
            (60, 20),  # Frame 4
        ]

    def move(self, keys):
        # Horizontal movement
        if keys[pygame.K_a ]:
            self.velocity_x = -self.move_speed
            self.facing_right = False
        elif keys[pygame.K_d]:
            self.velocity_x = self.move_speed
            self.facing_right = True
        
        # Apply friction
        if self.on_ground:
            self.velocity_x *= self.ground_friction
        else:
            self.velocity_x *= self.air_friction

        # Jumping
        if keys[pygame.K_SPACE] and self.on_ground:
            self.velocity_y = self.jump_speed
            self.on_ground = False

        # Apply gravity
        if not self.on_ground:
            self.velocity_y += self.gravity
            if self.velocity_y > self.max_fall_speed:
                self.velocity_y = self.max_fall_speed

        # Update position
        self.x += self.velocity_x
        self.y += self.velocity_y

        # Ground check
        if self.y > WINDOW_HEIGHT - self.height:
            self.y = WINDOW_HEIGHT - self.height
            self.velocity_y = 0
            self.on_ground = True

    def update_attack_animation(self):
        if self.attacking:
            if not self.attack_animation_active:
                self.attack_animation_active = True
                self.attack_frame = 0
                self.attack_animation_timer = 0
            
            self.attack_animation_timer += 1/60
            if self.attack_animation_timer >= self.attack_frame_duration:
                self.attack_frame = (self.attack_frame + 1) % self.attack_animation_frames
                self.attack_animation_timer = 0
                
                if self.attack_frame >= self.attack_animation_frames - 1:
                    self.attacking = False
                    self.attack_animation_active = False
        else:
            self.attack_animation_active = False

    def update_hitboxes(self):
        mouse_x, mouse_y = pygame.mouse.get_pos()
        dx = mouse_x - self.x
        dy = mouse_y - self.y
        
        if abs(dx) > abs(dy):
            if dx > 0:
                self.attack_direction = "right"
                self.attack_hitbox = pygame.Rect(self.x + self.width, self.y + 10, 60, 20)
                self.block_hitbox = pygame.Rect(self.x + self.width, self.y, 30, 40)
            else:
                self.attack_direction = "left"
                self.attack_hitbox = pygame.Rect(self.x - 60, self.y + 10, 60, 20)
                self.block_hitbox = pygame.Rect(self.x - 30, self.y, 30, 40)
        else:
            if dy > 0:
                self.attack_direction = "down"
                self.attack_hitbox = pygame.Rect(self.x + 10, self.y + self.height, 20, 60)
                self.block_hitbox = pygame.Rect(self.x, self.y + self.height, 40, 30)
            else:
                self.attack_direction = "up"
                self.attack_hitbox = pygame.Rect(self.x + 10, self.y - 60, 20, 60)
                self.block_hitbox = pygame.Rect(self.x, self.y - 30, 40, 30)

    def draw(self, screen):
        player_color = BLUE
        self.update_hitboxes()
        self.update_attack_animation()
        
        # Visual effect for dodging
        if self.is_dodging:
            player_color = (100, 100, 255)
            for i in range(3):
                trail_x = self.x - (self.velocity_x * i * 0.2)
                alpha = 150 - (i * 50)
                trail_surface = pygame.Surface((self.width, self.height), pygame.SRCALPHA)
                pygame.draw.rect(trail_surface, (*player_color, alpha), (0, 0, self.width, self.height))
                screen.blit(trail_surface, (trail_x, self.y))

        # Draw attack animation
        if self.attacking:
            width, height = self.attack_sizes[self.attack_frame]
            arc_radius = 60
            arc_angle = (self.attack_frame / self.attack_animation_frames) * 90
            
            if self.attack_direction == "right":
                arc_x = self.x + self.width + math.cos(math.radians(arc_angle)) * arc_radius
                arc_y = self.y + (self.height/2) - math.sin(math.radians(arc_angle)) * arc_radius
                pygame.draw.rect(screen, (255, 0, 0, 128), (arc_x, arc_y, width, height))
            elif self.attack_direction == "left":
                arc_x = self.x - math.cos(math.radians(arc_angle)) * arc_radius
                arc_y = self.y + (self.height/2) - math.sin(math.radians(arc_angle)) * arc_radius
                pygame.draw.rect(screen, (255, 0, 0, 128), (arc_x - width, arc_y, width, height))
            elif self.attack_direction == "up":
                arc_x = self.x + (self.width/2) + math.sin(math.radians(arc_angle)) * arc_radius
                arc_y = self.y - math.cos(math.radians(arc_angle)) * arc_radius
                pygame.draw.rect(screen, (255, 0, 0, 128), (arc_x - height/2, arc_y - width, height, width))
            elif self.attack_direction == "down":
                arc_x = self.x + (self.width/2) + math.sin(math.radians(arc_angle)) * arc_radius
                arc_y = self.y + self.height + math.cos(math.radians(arc_angle)) * arc_radius
                pygame.draw.rect(screen, (255, 0, 0, 128), (arc_x - height/2, arc_y, height, width))

        # Visual effect for blocking
        if self.blocking:
            pygame.draw.rect(screen, YELLOW, (self.x - 5, self.y - 5, self.width + 10, self.height + 10), 2)
            pygame.draw.rect(screen, (255, 255, 0, 128), self.block_hitbox, 2)
            
        # Draw player character
        pygame.draw.rect(screen, player_color, (self.x, self.y, self.width, self.height))
        
        # Draw health and posture bars
        pygame.draw.rect(screen, RED, (10, 10, self.health, 20))
        pygame.draw.rect(screen, YELLOW, (10, 40, self.posture, 20))

class Boss:
    def __init__(self):
        self.x = 600
        self.y = 300
        self.width = 50
        self.height = 50
        self.health = 100
        self.posture = 0
        self.attack_timer = 0
        self.attacking = False
        self.velocity_y = 0
        self.gravity = 0.8
        self.on_ground = False
        
        # Combat range settings
        self.ideal_distance = 120
        self.weapon_reach = 80
        self.weapon_hitbox = pygame.Rect(0, 0, self.weapon_reach, 20)
        self.facing_right = True
        
    def update(self, player):
        # Calculate distance to player
        dx = player.x - self.x
        dy = player.y - self.y
        distance = math.sqrt(dx*dx + dy*dy)
        
        # Update facing direction
        self.facing_right = dx > 0
        
        # Boss AI and movement
        if self.attack_timer <= 0:
            self.attacking = True
            self.attack_timer = 60
        else:
            self.attack_timer -= 1
            self.attacking = False
            
        # Movement AI with distance management
        if not self.attacking:
            if distance < self.ideal_distance - 20:  # Too close
                self.x -= dx/distance * 3
                self.y -= dy/distance * 3
            elif distance > self.ideal_distance + 20:  # Too far
                self.x += dx/distance * 3
                self.y += dy/distance * 3
                
        # Update weapon hitbox based on facing direction
        if self.facing_right:
            self.weapon_hitbox = pygame.Rect(self.x + self.width, self.y + 15, self.weapon_reach, 20)
        else:
            self.weapon_hitbox = pygame.Rect(self.x - self.weapon_reach, self.y + 15, self.weapon_reach, 20)
        
        # Apply gravity
        if not self.on_ground:
            self.velocity_y += self.gravity
        
        self.y += self.velocity_y
        
        # Ground check
        if self.y > WINDOW_HEIGHT - self.height:
            self.y = WINDOW_HEIGHT - self.height
            self.velocity_y = 0
            self.on_ground = True

    def draw(self, screen):
        # Draw boss body
        color = RED if self.attacking else (150, 0, 0)
        pygame.draw.rect(screen, color, (self.x, self.y, self.width, self.height))
        
        # Draw weapon hitbox when attacking
        if self.attacking:
            pygame.draw.rect(screen, (255, 0, 0, 128), self.weapon_hitbox, 2)
        
        # Draw boss health and posture bars
        pygame.draw.rect(screen, RED, (WINDOW_WIDTH-110, 10, self.health, 20))
        pygame.draw .rect(screen, YELLOW, (WINDOW_WIDTH-110, 40, self.posture, 20))

def main():
    player = Player()
    boss = Boss()
    running = True
    
    while running:
        screen.fill(BLACK)
        
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                running = False
            if event.type == pygame.MOUSEBUTTONDOWN:
                if event.button == 1:  # Left click to attack
                    player.attacking = True
                if event.button == 3:  # Right click to block
                    player.blocking = True
            if event.type == pygame.MOUSEBUTTONUP:
                if event.button == 1:
                    player.attacking = False
                if event.button == 3:
                    player.blocking = False
        
        # Update
        keys = pygame.key.get_pressed()
        player.move(keys)
        boss.update(player)
        
        # Combat logic
        if player.attacking and abs(player.x - boss.x) < 60 and abs(player.y - boss.y) < 60:
            boss.health -= 1
            boss.posture += 2
            
        if boss.attacking and abs(player.x - boss.x) < 60 and abs(player.y - boss.y) < 60:
            if player.blocking:
                boss.posture += 5
                # Deflect effect
                pygame.draw.circle(screen, YELLOW, (int((player.x + boss.x)/2), int((player.y + boss.y)/2)), 20)
            else:
                player.health -= 2
                player.posture += 3
        
        # Draw
        player.draw(screen)
        boss.draw(screen)
        
        # Display combat instructions
        font = pygame.font.Font(None, 36)
        text = font.render("WASD: Move, LMB: Attack, RMB: Block, SPACE: Jump", True, WHITE)
        screen.blit(text, (10, WINDOW_HEIGHT - 40))
        
        pygame.display.flip()
        clock.tick(60)

    pygame.quit()

if __name__ == "__main__":
    main()
