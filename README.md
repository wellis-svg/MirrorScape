import pygame
import random
import sys
import math
import json
import os
import logging
import heapq  # For A* pathfinding

# Initialize logging
logging.basicConfig(filename='mirrorscape.log', level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Initialize Pygame
try:
    pygame.init()
    pygame.mixer.init()
except Exception as e:
    logging.error(f"Pygame init failed: {e}")
    sys.exit(1)

# Constants
SCREEN_WIDTH = 800
SCREEN_HEIGHT = 600
CELL_SIZE = 40
MAZE_WIDTH = SCREEN_WIDTH // CELL_SIZE
MAZE_HEIGHT = SCREEN_HEIGHT // CELL_SIZE
FPS = 60
SAVE_FILE = 'mirrorscape_save.json'
HIGH_SCORE_FILE = 'high_scores.json'

# Colors
BLACK = (0, 0, 0)
WHITE = (255, 255, 255)
RED = (255, 0, 0)
GREEN = (0, 255, 0)
BLUE = (0, 0, 255)
YELLOW = (255, 255, 0)
PURPLE = (128, 0, 128)
ORANGE = (255, 165, 0)
CYAN = (0, 255, 255)
GRAY = (128, 128, 128)

# Load assets with fallbacks
try:
    attack_sound = pygame.mixer.Sound('sounds/attack.wav')
    collect_sound = pygame.mixer.Sound('sounds/collect.wav')
    damage_sound = pygame.mixer.Sound('sounds/damage.wav')
    win_sound = pygame.mixer.Sound('sounds/win.wav')
    bg_music = 'sounds/bg_music.mp3'
    pygame.mixer.music.load(bg_music)
    pygame.mixer.music.set_volume(0.5)
    pygame.mixer.music.play(-1)
except:
    attack_sound = collect_sound = damage_sound = win_sound = bg_music = None
    logging.warning("Audio assets not found; sounds disabled.")

# Sprite placeholders (use images in real implementation)
try:
    player_sprite = pygame.image.load('sprites/player.png').convert_alpha()
    enemy_sprite = pygame.image.load('sprites/enemy.png').convert_alpha()
except:
    player_sprite = enemy_sprite = None

# Particle class
class Particle:
    def __init__(self, x, y, color, lifetime=30, velocity=(0, -1)):
        self.x = x
        self.y = y
        self.color = color
        self.lifetime = lifetime
        self.velocity = velocity
        self.size = random.randint(2, 5)

    def update(self):
        self.lifetime -= 1
        self.x += self.velocity[0]
        self.y += self.velocity[1]
        return self.lifetime > 0

    def draw(self, screen):
        pygame.draw.circle(screen, self.color, (int(self.x), int(self.y)), self.size)

# Inventory class
class Inventory:
    def __init__(self):
        self.items = {'sword': 1, 'bow': 0, 'key': 0, 'potion': 0, 'bomb': 0}
        self.open = False
        self.selected = 0

    def add_item(self, item):
        if item in self.items:
            self.items[item] += 1

    def use_item(self, item):
        if self.items.get(item, 0) > 0:
            self.items[item] -= 1
            return True
        return False

    def draw(self, screen, font):
        if not self.open:
            return
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 128))
        screen.blit(overlay, (0, 0))
        y = 100
        for i, (item, count) in enumerate(self.items.items()):
            color = YELLOW if i == self.selected else WHITE
            text = font.render(f"{item.capitalize()}: {count}", True, color)
            screen.blit(text, (SCREEN_WIDTH // 2 - 100, y))
            y += 30

# Quest class
class Quest:
    def __init__(self, description, objective, reward):
        self.description = description
        self.objective = objective
        self.progress = 0
        self.reward = reward
        self.completed = False

    def update_progress(self, amount=1):
        self.progress += amount
        if self.progress >= self.objective:
            self.completed = True

    def draw(self, screen, font, y):
        color = GREEN if self.completed else WHITE
        text = font.render(f"{self.description}: {self.progress}/{self.objective}", True, color)
        screen.blit(text, (10, y))

# Achievement class
class Achievement:
    def __init__(self, name, description, condition):
        self.name = name
        self.description = description
        self.condition = condition
        self.unlocked = False

    def check_unlock(self, stats):
        if not self.unlocked and eval(self.condition, {"stats": stats}):
            self.unlocked = True
            return True
        return False

# Player class (expanded)
class Player:
    def __init__(self, x, y, player_id=1):
        self.x = x
        self.y = y
        self.player_id = player_id
        self.speed = 5
        self.weapon = 'sword'
        self.weapon_level = 1
        self.health = 100
        self.max_health = 100
        self.flashlight_on = False
        self.inventory = Inventory()
        self.rect = pygame.Rect(x * CELL_SIZE, y * CELL_SIZE, CELL_SIZE, CELL_SIZE)
        self.particles = []
        self.animation_frame = 0
        self.facing = 'down'

    def move(self, dx, dy, maze):
        new_x = self.x + dx
        new_y = self.y + dy
        if 0 <= new_x < MAZE_WIDTH and 0 <= new_y < MAZE_HEIGHT:
            cell = maze[new_y][new_x]
            if cell == 1 or (cell == 4 and not self.inventory.items['key'] > 0):  # Door without key
                return
            if cell == 3:  # Trap
                self.take_damage(10)
            self.x = new_x
            self.y = new_y
            self.rect.topleft = (self.x * CELL_SIZE, self.y * CELL_SIZE)
            self.update_facing(dx, dy)

    def update_facing(self, dx, dy):
        if dx > 0:
            self.facing = 'right'
        elif dx < 0:
            self.facing = 'left'
        elif dy > 0:
            self.facing = 'down'
        elif dy < 0:
            self.facing = 'up'

    def attack(self, enemies, projectiles, maze):
        if self.weapon == 'sword':
            for enemy in enemies[:]:
                if self.rect.colliderect(enemy.rect):
                    enemy.health -= 10 * self.weapon_level
                    self.particles.append(Particle(self.rect.centerx, self.rect.centery, RED))
                    if enemy.health <= 0:
                        enemies.remove(enemy)
                        for _ in range(10):
                            self.particles.append(Particle(enemy.rect.centerx, enemy.rect.centery, ORANGE))
                    if attack_sound:
                        attack_sound.play()
        elif self.weapon == 'bow':
            dir_map = {'up': (0, -1), 'down': (0, 1), 'left': (-1, 0), 'right': (1, 0)}
            dx, dy = dir_map.get(self.facing, (0, -1))
            projectiles.append(Projectile(self.x, self.y, dx, dy, 'player'))
            if attack_sound:
                attack_sound.play()
        elif self.weapon == 'bomb' and self.inventory.use_item('bomb'):
            # Place bomb
            projectiles.append(Projectile(self.x, self.y, 0, 0, 'bomb'))

    def take_damage(self, amount):
        self.health -= amount
        if self.health <= 0:
            self.health = 0
        self.particles.append(Particle(self.rect.centerx, self.rect.centery, RED))
        if damage_sound:
            damage_sound.play()

    def upgrade_weapon(self, new_weapon):
        self.weapon = new_weapon
        self.weapon_level += 1

    def update_particles(self):
        self.particles = [p for p in self.particles if p.update()]

    def draw_particles(self, screen):
        for p in self.particles:
            p.draw(screen)

    def draw(self, screen):
        if player_sprite:
            # Animated sprite (placeholder)
            frame = (self.animation_frame // 10) % 4
            screen.blit(player_sprite, self.rect, (frame * CELL_SIZE, 0, CELL_SIZE, CELL_SIZE))
        else:
            pygame.draw.rect(screen, GREEN, self.rect)

# Enemy classes (expanded with AI)
class Enemy:
    def __init__(self, x, y, type_='chaser'):
        self.x = x
        self.y = y
        self.type = type_
        self.health = 50 if type_ == 'chaser' else 30 if type_ == 'ranged' else 200
        self.speed = 2 if type_ == 'chaser' else 1 if type_ == 'ranged' else 3
        self.damage = 10 if type_ != 'boss' else 20
        self.rect = pygame.Rect(x * CELL_SIZE, y * CELL_SIZE, CELL_SIZE, CELL_SIZE)
        self.last_attack = 0
        self.particles = []
        self.path = []
        self.phase = 1 if type_ == 'boss' else 0

    def a_star_path(self, start, goal, maze):
        def heuristic(a, b):
            return abs(a[0] - b[0]) + abs(a[1] - b[1])
        frontier = []
        heapq.heappush(frontier, (0, start))
        came_from = {start: None}
        cost_so_far = {start: 0}
        while frontier:
            current = heapq.heappop(frontier)[1]
            if current == goal:
                break
            for dx, dy in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
                next_pos = (current[0] + dx, current[1] + dy)
                if 0 <= next_pos[0] < MAZE_WIDTH and 0 <= next_pos[1] < MAZE_HEIGHT and maze[next_pos[1]][next_pos[0]] != 1:
                    new_cost = cost_so_far[current] + 1
                    if next_pos not in cost_so_far or new_cost < cost_so_far[next_pos]:
                        cost_so_far[next_pos] = new_cost
                        priority = new_cost + heuristic(goal, next_pos)
                        heapq.heappush(frontier, (priority, next_pos))
                        came_from[next_pos] = current
        path = []
        current = goal
        while current != start:
            path.append(current)
            current = came_from.get(current)
            if current is None:
                return []
        path.reverse()
        return path

    def move_towards_player(self, player, maze):
        if self.type == 'chaser':
            if not self.path or random.random() < 0.1:
                self.path = self.a_star_path((int(self.x), int(self.y)), (int(player.x), int(player.y)), maze)
            if self.path:
                next_pos = self.path.pop(0)
                self.x, self.y = next_pos
                self.rect.topleft = (self.x * CELL_SIZE, self.y * CELL_SIZE)
        elif self.type == 'ranged':
            if random.random() < 0.01:
                dx = 1 if player.x > self.x else -1 if player.x < self.x else 0
                dy = 1 if player.y > self.y else -1 if player.y < self.y else 0
                projectiles.append(Projectile(self.x, self.y, dx, dy, 'enemy'))
        elif self.type == 'boss':
            if self.health < 100 and self.phase == 1:
                self.phase = 2
                self.speed = 5
            self.x += random.choice([-1, 0, 1])
            self.y += random.choice([-1, 0, 1])
            self.rect.topleft = (self.x * CELL_SIZE, self.y * CELL_SIZE)

    def attack_player(self, player):
        if self.rect.colliderect(player.rect) and pygame.time.get_ticks() - self.last_attack > 1000:
            player.take_damage(self.damage)
            self.last_attack = pygame.time.get_ticks()

    def update_particles(self):
        self.particles = [p for p in self.particles if p.update()]

    def draw_particles(self, screen):
        for p in self.particles:
            p.draw(screen)

    def draw(self, screen):
        if enemy_sprite:
            screen.blit(enemy_sprite, self.rect)
        else:
            color = RED if self.type == 'chaser' else PURPLE if self.type == 'ranged' else ORANGE
            pygame.draw.rect(screen, color, self.rect)

# Projectile class (expanded)
class Projectile:
    def __init__(self, x, y, dx, dy, owner):
        self.x = x
        self.y = y
        self.dx = dx
        self.dy = dy
        self.owner = owner
        self.speed = 10
        self.rect = pygame.Rect(x * CELL_SIZE, y * CELL_SIZE, 10, 10)
        self.trail = []
        self.explode_timer = 60 if owner == 'bomb' else 0

    def move(self, maze):
        if self.owner == 'bomb':
            self.explode_timer -= 1
            if self.explode_timer <= 0:
                # Explosion effect
                for _ in range(20):
                    particles.append(Particle(self.x * CELL_SIZE, self.y * CELL_SIZE, ORANGE))
                # Damage nearby
                for enemy in enemies[:]:
                    if math.dist((self.x, self.y), (enemy.x, enemy.y)) < 2:
                        enemy.health -= 50
                        if enemy.health <= 0:
                            enemies.remove(enemy)
                return False
        self.x += self.dx * self.speed / FPS
        self.y += self.dy * self.speed / FPS
        self.rect.center = (self.x * CELL_SIZE, self.y * CELL_SIZE)
        self.trail.append((self.x, self.y))
        if len(self.trail) > 10:
            self.trail.pop(0)
        if maze[int(self.y)][int(self.x)] == 2:
            self.dx *= -1
            self.dy *= -1
        if not (0 <= self.x < MAZE_WIDTH and 0 <= self.y < MAZE_HEIGHT) or maze[int(self.y)][int(self.x)] == 1:
            return False
        return True

    def draw_trail(self, screen):
        for i, pos in enumerate(self.trail):
            alpha = 255 * (i / len(self.trail))
            color = (GREEN[0], GREEN[1], GREEN[2], alpha) if self.owner == 'player' else (RED[0], RED[1], RED[2], alpha)
            pygame.draw.circle(screen, color, (int(pos[0] * CELL_SIZE), int(pos[1] * CELL_SIZE)), 3)

# Power-up class (expanded)
class PowerUp:
    def __init__(self, x, y, type_):
        self.x = x
        self.y = y
        self.type = type_
        self.rect = pygame.Rect(x * CELL_SIZE, y * CELL_SIZE, CELL_SIZE, CELL_SIZE)
        self.glow = 0

    def update_glow(self):
        self.glow = (self.glow + 1) % 60

    def draw_glow(self, screen):
        intensity = abs(math.sin(self.glow * math.pi / 30)) * 100
        color = (YELLOW[0] + intensity, YELLOW[1], YELLOW[2] + intensity)
        pygame.draw.circle(screen, color, self.rect.center, CELL_SIZE // 2 + 5)

# Settings class
class Settings:
    def __init__(self):
        self.volume = 0.5
        self.difficulty = 'normal'
        self.controls = {'move_up': pygame.K_w, 'move_down': pygame.K_s, 'move_left': pygame.K_a, 'move_right': pygame.K_d, 'attack': pygame.K_SPACE}

    def draw_menu(self, screen, font):
        overlay = pygame.Surface((SCREEN_WIDTH, SCREEN_HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 128))
        screen.blit(overlay, (0, 0))
        y = 100
        text = font.render(f"Volume: {self.volume}", True, WHITE)
        screen.blit(text, (SCREEN_WIDTH // 2 - 100, y))
        y +=
