import os
import random
import sys
from time import time
import pygame


def load_image(name, color_key=-1):
    fullname = os.path.join('data', name)
    # если файл не существует, то выходим
    if not os.path.isfile(fullname):
        print(f"Файл с изображением '{fullname}' не найден")
        sys.exit()
    image = pygame.image.load(fullname)

    if color_key is not None:
        image = image.convert()
        if color_key == -1:
            color_key = image.get_at((0, 0))
        image.set_colorkey(color_key)
    else:
        image = image.convert_alpha()
    return image


def load_level(filename):
    filename = "data/" + filename
    # читаем уровень, убирая символы перевода строки
    with open(filename, 'r') as mapFile:
        level_map = [line.strip() for line in mapFile]

    # и подсчитываем максимальную длину
    max_width = max(map(len, level_map))

    # дополняем каждую строку пустыми клетками ('.')
    return list(map(lambda x: list(x.ljust(max_width, '.')), level_map))


def terminate():
    pygame.quit()
    sys.exit()


def start_screen():
    intro_text = ["ЗАСТАВКА", "",
                  "Правила игры",
                  "Если в правилах несколько строк,",
                  "приходится выводить их построчно"]

    fon = pygame.transform.scale(load_image('fon.jpg'), (WIDTH, HEIGHT))
    screen.blit(fon, (0, 0))
    font = pygame.font.Font(None, 30)
    text_coord = 50
    for line in intro_text:
        string_rendered = font.render(line, 1, pygame.Color('black'))
        intro_rect = string_rendered.get_rect()
        text_coord += 10
        intro_rect.top = text_coord
        intro_rect.x = 10
        text_coord += intro_rect.height
        screen.blit(string_rendered, intro_rect)

    while True:
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                terminate()
            elif event.type == pygame.KEYDOWN or \
                    event.type == pygame.MOUSEBUTTONDOWN:
                return  # начинаем игру
        pygame.display.flip()
        clock.tick(FPS)


class Tile(pygame.sprite.Sprite):
    def __init__(self, tile_type, pos_x, pos_y):

        # проверяем на "твердый ли" объект:
        if tile_type != "wall":
            super().__init__(tiles_group)
        else:
            super().__init__(solid_objects)
        self.image = tile_images[tile_type]
        self.pos_x = pos_x
        self.pos_y = pos_y
        self.rect = self.image.get_rect().move(
            tile_width * pos_x, tile_height * pos_y)


class Bullet(pygame.sprite.Sprite):
    def __init__(self, x, y, storona, type):
        if type == "player":
            pygame.sprite.Sprite.__init__(self)
            self.image = pygame.Surface((10, 20))
            self.image.fill('yellow')
        elif type == "enemy":
            pygame.sprite.Sprite.__init__(self)
            self.image = pygame.Surface((10, 20))
            self.image.fill('red')

        self.rect = self.image.get_rect()
        self.rect.bottom = y
        self.rect.centerx = x
        self.speedy = -10
        self.storona = storona

    def update(self):
        #self.rect.y += self.speedy
        if self.storona == 'u':
            self.rect.y += self.speedy
        if self.storona == 'd':
            self.rect.y -= self.speedy
        if self.storona == 'r':
            self.rect.x -= self.speedy
        if self.storona == 'l':
            self.rect.x += self.speedy


class Player(pygame.sprite.Sprite):
    def __init__(self, pos_x, pos_y):
        super().__init__(player_group)
        self.image = player_image
        self.rect = self.image.get_rect().move(
            tile_width * pos_x + 15, tile_height * pos_y + 5)
        self.pos = (pos_x, pos_y)

    def move(self, x, y):
        camera.dx -= tile_width * (x - self.pos[0])
        camera.dy -= tile_height * (y - self.pos[1])
        level_map[self.pos[1]][self.pos[0]] = "."
        self.pos = (x, y)
        level_map[self.pos[1]][self.pos[0]] = "@"

        # смещение спрайтов
        for sprite in tiles_group:
            camera.apply(sprite)
        for sprite in solid_objects:
            camera.apply(sprite)
        for sprite in enemies_group:
            camera.apply(sprite)
        for sprite in player_bullets:
            camera.apply(sprite)
        for sprite in enemy_bullets:
            camera.apply(sprite)

    def shoot(self, storona):
        bullet = Bullet(self.rect.centerx, self.rect.top + 30, storona, "player")
        all_sprites.add(bullet)
        player_bullets.add(bullet)


class Enemy(pygame.sprite.Sprite):
    def __init__(self, pos_x, pos_y):
        super().__init__(enemies_group)
        self.image = enemies_image
        self.pos_x = pos_x
        self.pos_y = pos_y
        self.storona = "l"
        self.rect = self.image.get_rect().move(
            tile_width * self.pos_x + 15, tile_height * self.pos_y + 5)
        self.pos = (self.pos_x, self.pos_y)

    def move(self, x, y):
        level_map[self.pos[1]][self.pos[0]] = "."
        self.pos_x = x
        self.pos_y = y
        self.storona = "l"
        self.rect = self.image.get_rect().move(
            tile_width * self.pos_x + 15, tile_height * self.pos_y + 5)
        self.pos = (self.pos_x, self.pos_y)

        level_map[self.pos[1]][self.pos[0]] = "&"

    def brain(self):
        action = random.randint(0, 4)
        action = 2
        return action

    def shoot(self, storona):
        bullet = Bullet(self.rect.centerx, self.rect.top + 30, storona, "enemy")
        all_sprites.add(bullet)
        enemy_bullets.add(bullet)



class Camera:
    # зададим начальный сдвиг камеры
    def __init__(self):
        self.dx = 0
        self.dy = 0

    # сдвинуть объект obj на смещение камеры
    def apply(self, obj):
        obj.rect.x += self.dx
        obj.rect.y += self.dy

    # позиционировать камеру на объекте target
    def update(self, target):
        self.dx = 0
        self.dy = 0


def generate_level(level):
    new_player, x, y = None, None, None
    for y in range(len(level)):
        for x in range(len(level[y])):
            # не знаю как оформить проверку на то, является ли объект твердым, пускай будет пока тут:
            if level[y][x] == '.':
                Tile('empty', x, y)
            elif level[y][x] == '#':
                Tile('wall', x, y)
            elif level[y][x] == '@':
                Tile('empty', x, y)
                new_player = Player(x, y)
            elif level[y][x] == "&":
                Tile('empty', x, y)
                new_enemy = Enemy(x, y)
    # вернем игрока, а также размер поля в клетках
    return new_player, x, y


def move(player, movement):
    x, y = player.pos
    if movement == "up":
        if y > 0 and level_map[y - 1][x] == ".":
            player.move(x, y - 1)
    elif movement == "down":
        if y < max_y and level_map[y + 1][x] == ".":
            player.move(x, y + 1)
    elif movement == "left":
        if x > 0 and level_map[y][x - 1] == ".":
            player.move(x - 1, y)
    elif movement == "right":
        if x < max_x and level_map[y][x + 1] == ".":
            player.move(x + 1, y)


pygame.init()
size = WIDTH, HEIGHT = (800, 800)
pygame.display.set_caption("Марио")
screen = pygame.display.set_mode(size)
FPS = 50
clock = pygame.time.Clock()
my_time = 0

# создание групп спрайтов:
tiles_group = pygame.sprite.Group()
player_group = pygame.sprite.Group()
player_bullets = pygame.sprite.Group()
enemy_bullets = pygame.sprite.Group()
all_sprites = pygame.sprite.Group()
solid_objects = pygame.sprite.Group()
enemies_group = pygame.sprite.Group()

# список столкновений объектов разных групп спрайтов:

collision_list = [(solid_objects, player_bullets, False, True), (enemies_group, player_bullets, True, True), (player_group, enemy_bullets, False, True)]

tile_images = {
    'wall': load_image('box.png'),
    'empty': load_image('grass2.png')
}

player_image = load_image('mario.png')
enemies_image = load_image("mario.png")

tile_width = tile_height = 50

start_screen()
camera = Camera()
level_map = load_level("map.map")
player, max_x, max_y = generate_level(level_map)
running = True
storona = 'u'

while running:
    tic = time()
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_SPACE:
                player.shoot(storona)
            if event.key == pygame.K_w:
                move(player, "up")
                storona = 'u'
            elif event.key == pygame.K_s:
                move(player, "down")
                storona = 'd'
            elif event.key == pygame.K_a:
                move(player, "left")
                storona = 'l'
            elif event.key == pygame.K_d:
                move(player, "right")
                storona = 'r'

    # каждые n количество секунд срабатывает рандомайзер для действий:
    if my_time > 5:
        my_time = 0
        for enem in enemies_group.sprites():
            action = enem.brain()
            x, y = enem.pos
            if action == 0:
                if level_map[y - 1][x] == ".":
                    enem.storona = 'u'
                    enem.move(x, y - 1)
            elif action == 1:
                if level_map[y + 1][x] == ".":
                    enem.storona = 'd'
                    enem.move(x, y + 1)
            elif action == 2:
                if level_map[y][x - 1] == ".":
                    enem.storona = 'l'
                    enem.move(x - 1, y)
            elif action == 3:
                if level_map[y][x + 1] == ".":
                    enem.storona = 'r'
                    enem.move(x + 1, y)
            elif action == 4:
                enem.shoot(enem.storona)

    screen.fill(pygame.Color("black"))
    camera.update(player)
    tiles_group.draw(screen)
    solid_objects.draw(screen)
    player_group.draw(screen)
    enemies_group.draw(screen)

    # собственно проверка на столкновение:
    for test in collision_list:
        hits = pygame.sprite.groupcollide(test[0], test[1], test[2], test[3])
        if test[0] == enemies_group:
            for hit in hits:
                print(level_map[hit.pos_y][hit.pos_x])
                level_map[hit.pos_y][hit.pos_x] = "."


    player_bullets.draw(screen)
    player_bullets.update()

    enemy_bullets.draw(screen)
    enemy_bullets.update()

    pygame.display.flip()
    clock.tick(FPS)

    toc = time()
    my_time += toc - tic

pygame.display.quit()
pygame.quit()
