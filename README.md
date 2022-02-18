# tutorial-to-pygames
import os
import sys
import pygame as pg


CAPTION = "Moving Platforms"
SCREEN_SIZE = (700,500)


class _Physics(object):
   
    def __init__(self):
        
        self.x_vel = self.y_vel = 0
        self.grav = 0.4
        self.fall = False

    def physics_update(self):
        
        if self.fall:
            self.y_vel += self.grav
        else:
            self.y_vel = 0


class Player(_Physics, pg.sprite.Sprite):
   
    def __init__(self,location,speed):
         _Physics.__init__(self)
         pg.sprite.Sprite.__init__(self)
         self.image = pg.Surface((30, 55)).convert()
         self.image.fill(pg.Color("red"))
         self.rect = self.image.get_rect(topleft=location)
         self.speed = speed
         self.jump_power = -9.0
         self.jump_cut_magnitude = -3.0
         self.on_moving = False
         self.collide_below = False

    def check_keys(self, keys):
        
        self.x_vel = 0
        if keys[pg.K_LEFT] or keys[pg.K_a]:
            self.x_vel -= self.speed
            print("left")
        if keys[pg.K_RIGHT] or keys[pg.K_d]:
            self.x_vel += self.speed
            print("right")

    def get_position(self, obstacles):
        
        if not self.fall:
            self.check_falling(obstacles)
        else:
            self.fall = self.check_collisions((0,self.y_vel), 1, obstacles)
        if self.x_vel:
            self.check_collisions((self.x_vel,0), 0, obstacles)

    def check_falling(self, obstacles):
       
        if not self.collide_below:
            self.fall = True
            self.on_moving = False

    def check_moving(self,obstacles):
       
        if not self.fall:
            now_moving = self.on_moving
            any_moving, any_non_moving = [], []
            for collide in self.collide_below:
                if collide.type == "moving":
                    self.on_moving = collide
                    any_moving.append(collide)
                else:
                    any_non_moving.append(collide)
            if not any_moving:
                self.on_moving = False
            elif any_non_moving or now_moving in any_moving:
                self.on_moving = now_moving

    def check_collisions(self, offset, index, obstacles):
        
        unaltered = True
        self.rect[index] += offset[index]
        while pg.sprite.spritecollideany(self, obstacles):
            self.rect[index] += (1 if offset[index]<0 else -1)
            unaltered = False
        return unaltered

    def check_above(self, obstacles):
        
        self.rect.move_ip(0, -1)
        collide = pg.sprite.spritecollideany(self, obstacles)
        self.rect.move_ip(0, 1)
        return collide

    def check_below(self, obstacles):
        
        self.rect.move_ip((0,1))
        collide = pg.sprite.spritecollide(self, obstacles, False)
        self.rect.move_ip((0,-1))
        return collide

    def jump(self, obstacles):
        
        if not self.fall and not self.check_above(obstacles):
            self.y_vel = self.jump_power
            self.fall = True
            self.on_moving = False
            

    def jump_cut(self):
        
        if self.fall:
            if self.y_vel < self.jump_cut_magnitude:
                self.y_vel = self.jump_cut_magnitude
                print("small jump")
            else:
                print("jump")

    def pre_update(self, obstacles):
        
        self.collide_below = self.check_below(obstacles)
        self.check_moving(obstacles)

    def update(self, obstacles, keys):
        
        self.check_keys(keys)
        self.get_position(obstacles)
        self.physics_update()

    def draw(self, surface):
        
        surface.blit(self.image, self.rect)


class Block(pg.sprite.Sprite):
   
    def __init__(self, color, rect):
        
        pg.sprite.Sprite.__init__(self)
        self.rect = pg.Rect(rect)
        self.image = pg.Surface(self.rect.size).convert()
        self.image.fill(color)
        self.type = "normal"


class MovingBlock(Block):
    
    def __init__(self, color, rect, end, axis, delay=500, speed=2, start=None):
           Block.__init__(self, color, rect)
           self.start = self.rect[axis]
           if start:
              self.rect[axis] = start
              self.axis = axis
              self.end = end
              self.timer = 0.0
              self.delay = delay
              self.speed = speed
              self.waiting = False
              self.type = "moving"

    def update(self, player, obstacles):
        
        obstacles = obstacles.copy()
        obstacles.remove(self)
        now = pg.time.get_ticks()
        if not self.waiting:
            speed = self.speed
            start_passed = self.start >= self.rect[self.axis]+speed
            end_passed = self.end <= self.rect[self.axis]+speed
            if start_passed or end_passed:
                if start_passed:
                    speed = self.start-self.rect[self.axis]
                else:
                    speed = self.end-self.rect[self.axis]
                self.change_direction(now)
            self.rect[self.axis] += speed
            self.move_player(now, player, obstacles, speed)
        elif now-self.timer > self.delay:
            self.waiting = False

    def move_player(self, now, player, obstacles, speed):
        
        if player.on_moving is self or pg.sprite.collide_rect(self,player):
            axis = self.axis
            offset = (speed, speed)
            player.check_collisions(offset, axis, obstacles)
            if pg.sprite.collide_rect(self, player):
                if self.speed > 0:
                    self.rect[axis] = player.rect[axis]-self.rect.size[axis]
                else:
                    self.rect[axis] = player.rect[axis]+player.rect.size[axis]
                self.change_direction(now)

    def change_direction(self, now):
      
        self.waiting = True
        self.timer = now
        self.speed *= -1


class Control(object):
    
    def __init__(self):
        
        self.screen = pg.display.get_surface()
        self.screen_rect = self.screen.get_rect()
        self.clock = pg.time.Clock()
        self.fps = 60.0
        self.keys = pg.key.get_pressed()
        self.done = False
        self.player = Player((50,875), 4)
        self.viewport = self.screen.get_rect()
        self.level = pg.Surface((1000,1000)).convert()
        self.level_rect = self.level.get_rect()
        self.win_text,self.win_rect = self.make_text()
        self.obstacles = self.make_obstacles()

    def make_text(self):
        
        font = pg.font.Font(None, 100)
        message = "You win. Celebrate."
        text = font.render(message, True, (100,100,175))
        rect = text.get_rect(centerx=self.level_rect.centerx, y=100)
        return text, rect

    def make_obstacles(self):
       
        walls = [Block(pg.Color("chocolate"), (0,980,1000,20)),
                 Block(pg.Color("chocolate"), (0,0,20,1000)),
                 Block(pg.Color("chocolate"), (980,0,20,1000))]
        static = [Block(pg.Color("darkgreen"), (250,780,200,100)),
                  Block(pg.Color("darkgreen"), (600,880,200,100)),
                  Block(pg.Color("darkgreen"), (20,360,880,40)),
                  Block(pg.Color("darkgreen"), (950,400,30,20)),
                  Block(pg.Color("darkgreen"), (20,630,50,20)),
                  Block(pg.Color("darkgreen"), (80,530,50,20)),
                  Block(pg.Color("darkgreen"), (130,470,200,215)),
                  Block(pg.Color("darkgreen"), (20,760,30,20)),
                  Block(pg.Color("darkgreen"), (400,740,30,40))]
        moving = [MovingBlock(pg.Color("olivedrab"), (20,740,75,20), 325, 0),
                  MovingBlock(pg.Color("olivedrab"), (600,500,100,20), 880, 0),
                  MovingBlock(pg.Color("olivedrab"),
                              (420,430,100,20), 550, 1, speed=3, delay=200),
                  MovingBlock(pg.Color("olivedrab"),
                              (450,700,50,20), 930, 1, start=930),
                  MovingBlock(pg.Color("olivedrab"),
                              (500,700,50,20), 730, 0, start=730),
                  MovingBlock(pg.Color("olivedrab"),
                              (780,700,50,20), 895, 0, speed=-1)]
        return pg.sprite.Group(walls, static, moving)

    def update_viewport(self):
     
        self.viewport.center = self.player.rect.center
        self.viewport.clamp_ip(self.level_rect)

    def event_loop(self):
       
        for event in pg.event.get():
            if event.type == pg.QUIT or self.keys[pg.K_ESCAPE]:
                self.done = True
            elif event.type == pg.KEYDOWN:
                if event.key == pg.K_SPACE:
                    self.player.jump(self.obstacles)
            elif event.type == pg.KEYUP:
                if event.key == pg.K_SPACE:
                    self.player.jump_cut()

    def update(self):
        
        self.keys = pg.key.get_pressed()
        self.player.pre_update(self.obstacles)
        self.obstacles.update(self.player, self.obstacles)
        self.player.update(self.obstacles, self.keys)
        self.update_viewport()

    def draw(self):
          self.level.fill(pg.Color("lightblue"))
          self.obstacles.draw(self.level)
          self.level.blit(self.win_text, self.win_rect)
          self.player.draw(self.level)
          self.screen.blit(self.level, (0,0), self.viewport)

    def display_fps(self):

        caption = "{} - FPS: {:.2f}".format(CAPTION, self.clock.get_fps())
        pg.display.set_caption(caption)

    def main_loop(self):
        
        while not self.done:
            self.event_loop()
            self.update()
            self.draw()
            pg.display.update()
            self.clock.tick(self.fps)
            self.display_fps()


if __name__ == "__main__":
    os.environ['SDL_VIDEO_CENTERED'] = '1'
    pg.init()
    pg.display.set_caption(CAPTION)
    pg.display.set_mode(SCREEN_SIZE)
    run_it = Control()
    run_it.main_loop()
    pg.quit()
    sys.exit()
