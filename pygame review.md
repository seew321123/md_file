# pygame review
### a simple project 
```python
import pygame

import sys

pygame.init()

screen = pygame.display.set_mode((1280,720))

pygame.display.set_caption('haha')

bc_color = (230,230,230)

while True:

	for event in pygame.event.get():
		if event.type == pygame.QUIT:
			sys.exit()
	screen.fill(bc_color)
	pygame.display.flip()
run.game()
!/(^[1-9]\d*$)/.test(state.mart_publish_count) || !/(^(0\.[0-9][0-9]?|[1-9]\d*|[1-9]\d*\.[0-9][0-9]?)$)/.test(state.mart_publish_price)
正则表达式判断正整数以及正小数或整数的方式


```