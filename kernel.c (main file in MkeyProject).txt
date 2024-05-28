/*
* Copyright (C) 2014  Arjun Sreedharan
* License: GPL version 2 or higher http://www.gnu.org/licenses/gpl.html
*/
#include "keyboard_map.h"

/* there are 25 lines each of 80 columns; each element takes 2 bytes */ 
//Boyutu 25x80 
#define LINES 25
#define COLUMNS_IN_LINE 80
#define BYTES_FOR_EACH_ELEMENT 2
#define SCREENSIZE BYTES_FOR_EACH_ELEMENT * COLUMNS_IN_LINE * LINES

#define KEYBOARD_DATA_PORT 0x60
#define KEYBOARD_STATUS_PORT 0x64
#define IDT_SIZE 256
#define INTERRUPT_GATE 0x8e
#define KERNEL_CODE_SEGMENT_OFFSET 0x08

#define ENTER_KEY_CODE 0x1C
#define LEFT_KEY_CODE 0x1E 
#define RIGHT_KEY_CODE 0x20 
#define SPACE_KEY_CODE 0x39
#define SIZE 15

int x = COLUMNS_IN_LINE / 2;
int y = 22;
int score = 0;
int enemyX = COLUMNS_IN_LINE / 2;
int enemyY = 0;
int gameover = 0;
int printbullet = 0;
int didStart = 0;
unsigned int random_arr_index = 0;
unsigned int random_arr_length = 100;
unsigned int MAX_BULLETS = 1000;
int level = 1;

//Bullet
struct Bullet {
    int x;
    int y;
    int active;
};

struct Bullet bullet[1000];


void gotoxy(unsigned int x, unsigned int y);
void draw_strxy(const char *str,unsigned int x, unsigned int y);
void draw_player(void);
void draw_enemy(void);
void gameboard(void);
void game(void);
void start_screen(void);
void end_screen(void);
void print_integer(int n,int color);
void clear_player_right(void);
void clear_player_left(void);
void clear_enemy_up(void);
int simple_rand(unsigned int random_arr_index,unsigned int random_arr_length);
int checkCollision(void);
void shoot_bullet(void);
void update_bullets(void);
void clear_enemy(void);

extern unsigned char keyboard_map[128];
extern void keyboard_handler(void);
extern char read_port(unsigned short port);
extern void write_port(unsigned short port, unsigned char data);
extern void load_idt(unsigned long *idt_ptr);

/* current cursor location */
unsigned int current_loc = 0;
/* video memory begins at address 0xb8000 */
char *vidptr = (char*)0xb8000;

struct IDT_entry {
	unsigned short int offset_lowerbits;
	unsigned short int selector;
	unsigned char zero;
	unsigned char type_attr;
	unsigned short int offset_higherbits;
};

struct IDT_entry IDT[IDT_SIZE];


void idt_init(void)
{
	unsigned long keyboard_address;
	unsigned long idt_address;
	unsigned long idt_ptr[2];

	/* populate IDT entry of keyboard's interrupt */
	keyboard_address = (unsigned long)keyboard_handler;
	IDT[0x21].offset_lowerbits = keyboard_address & 0xffff;
	IDT[0x21].selector = KERNEL_CODE_SEGMENT_OFFSET;
	IDT[0x21].zero = 0;
	IDT[0x21].type_attr = INTERRUPT_GATE;
	IDT[0x21].offset_higherbits = (keyboard_address & 0xffff0000) >> 16;

	/*     Ports
	*	 PIC1	PIC2
	*Command 0x20	0xA0
	*Data	 0x21	0xA1
	*/

	/* ICW1 - begin initialization */
	write_port(0x20 , 0x11);
	write_port(0xA0 , 0x11);

	/* ICW2 - remap offset address of IDT */
	/*
	* In x86 protected mode, we have to remap the PICs beyond 0x20 because
	* Intel have designated the first 32 interrupts as "reserved" for cpu exceptions
	*/
	write_port(0x21 , 0x20);
	write_port(0xA1 , 0x28);

	/* ICW3 - setup cascading */
	write_port(0x21 , 0x00);
	write_port(0xA1 , 0x00);

	/* ICW4 - environment info */
	write_port(0x21 , 0x01);
	write_port(0xA1 , 0x01);
	/* Initialization finished */

	/* mask interrupts */
	write_port(0x21 , 0xff);
	write_port(0xA1 , 0xff);

	/* fill the IDT descriptor */
	idt_address = (unsigned long)IDT ;
	idt_ptr[0] = (sizeof (struct IDT_entry) * IDT_SIZE) + ((idt_address & 0xffff) << 16);
	idt_ptr[1] = idt_address >> 16 ;

	load_idt(idt_ptr);
}

void kb_init(void)
{
	/* 0xFD is 11111101 - enables only IRQ1 (keyboard)*/
	write_port(0x21 , 0xFD);
}

void kprint(const char *str, int color)
{
	unsigned int i = 0;
	while (str[i] != '\0') {
		vidptr[current_loc++] = str[i++];
		vidptr[current_loc++] = color;
	}
}

void kprint_newline(void)
{
	unsigned int line_size = BYTES_FOR_EACH_ELEMENT * COLUMNS_IN_LINE;
	current_loc = current_loc + (line_size - current_loc % (line_size));
}

void gotoxy(unsigned int x, unsigned int y) // belirli koordinata gitmek icin hesaplama fonk.
{
	unsigned int line_size = BYTES_FOR_EACH_ELEMENT * COLUMNS_IN_LINE;
	current_loc = BYTES_FOR_EACH_ELEMENT * (x * COLUMNS_IN_LINE + y);
}

void draw_strxy(const char *str,unsigned int x, unsigned int y) // verilen koordinata yazi yazan fonk.
{
	gotoxy(y,x);
	kprint(str,0x0F);
}

void sleep(int sec) // bekletme fonk.
{
	int i;
  for(i=0;i<sec;i++);

}
void clear_screen(void)
{
	unsigned int i = 0;
	while (i < SCREENSIZE) {
		vidptr[i++] = ' ';
		vidptr[i++] = 0x07;
	}
}

void keyboard_handler_main(void)
{
	unsigned char status;
	char keycode;

	/* write EOI */
	write_port(0x20, 0x20);

	status = read_port(KEYBOARD_STATUS_PORT);
	/* Lowest bit of status will be set if buffer is not empty */
	if (status & 0x01) {
		keycode = read_port(KEYBOARD_DATA_PORT);
		if(keycode < 0)
			return;
		
		if(keycode == ENTER_KEY_CODE) {
			didStart = 1;
			return;
		}
		
		if(keycode == LEFT_KEY_CODE) {
		        if(x > 0 && didStart)
		        {
		          clear_player_right();
	                  x = x - 1;
	                  draw_player();
		        }
			return;
		}
		if(keycode == RIGHT_KEY_CODE) {
		        if(x < 73 && didStart)
		        {
		          clear_player_left();
	                  x = x + 1;
	                  draw_player();
		        }
			return; 
		}
		if(keycode == SPACE_KEY_CODE) {
		        if(didStart)
		        {
			shoot_bullet();
		        }
			return; 
		}
		else
		{
		return;
		}

		vidptr[current_loc++] = keyboard_map[(unsigned char) keycode];
		vidptr[current_loc++] = 0x07;
	}
}

void print_integer(int n, int color) // sayi print etme
{
    int i = 0;
    char buffer[20];  
    do {
        buffer[i++] = (n % 10) + '0';  
        n /= 10;
    } while (n != 0);
    while (i > 0) {
      vidptr[current_loc++] = buffer[--i];
      vidptr[current_loc++] = color;
    }
}

void end_screen(void) // bitis ekrani
{
draw_strxy("Game over!",1,1);
draw_strxy("Your Final Score: ",1,2);
print_integer(score,0x02);
draw_strxy("Press any key to play again . . .",1,3);
}

void start_screen(void) // baslangic ekrani
{
    kprint("Instructions:",0x0F);
     kprint_newline();
    kprint("- Try to kill the enemies and make the best score,",0x0F);
     kprint_newline();
    kprint("- If you touch them or if they pass you the game will over.",0x0F);
     kprint_newline();
    kprint("- Use 'a' to move left.",0x0F);
     kprint_newline();
    kprint("- Use 'd' to move right.",0x0F);
     kprint_newline();
    kprint("- Press Spacebar to shoot.",0x0F);
     kprint_newline();
     kprint_newline();
    kprint("Press Enter to start the game...",0x0F);
 
  while(!didStart);
  
  clear_screen();
}
void shoot_bullet() {
    // Find an inactive bullet in the array
    for (int i = 0; i < MAX_BULLETS; i++) {     
        if (!bullet[i].active) {
            // Activate the bullet and set its position
            bullet[i].x = x + 3;
            bullet[i].y = y - 1;
            bullet[i].active = 1;
            break; // Exit the loop after activating one bullet
        }
    }
}

void update_bullets() {
    for (int i = 0; i < MAX_BULLETS; i++) {
    	if(bullet[i].y < 0){
    	bullet[i].active = 0;
    	}
        if (bullet[i].active) {
            bullet[i].y -= 2; // Move bullet upward
            draw_strxy(".",bullet[i].x,bullet[i].y);
            draw_strxy(" ",bullet[i].x,bullet[i].y+2); // clears back of the bullet
        if ((bullet[i].x >= enemyX - 4 && bullet[i].x <= enemyX + 4) && (bullet[i].y >= enemyY && bullet[i].y <= enemyY + 3)) // check collision between bullet and enemy
        {
        	score++; // Increase score
        	clear_enemy();
        	bullet[i].active = 0;
        	draw_strxy(" ",bullet[i].x,bullet[i].y); // clears the bullet that has the collision
        	
        	enemyY = 0;
		enemyX = simple_rand(random_arr_index,random_arr_length); // generating random number
		random_arr_index += 1;
         }
        }
    }
}

int checkCollision(void)
{

   if ((x >= enemyX - 5 && x <= enemyX + 5) && (y >= enemyY && y <= enemyY + 2))
    {
        return 1; // End the game if collision detected between player and enemy
    }

return 0;
}

int simple_rand(unsigned int random_arr_index,unsigned int random_arr_length) {
    static unsigned int seed = 0xDEADBEEF;
    unsigned int a = 1103515245;
    unsigned int c = 12345;
    unsigned int m = 0x7fffffff;
    int random_array[random_arr_length];
    int arrayseed = 0;
    
    
    for(int i = 0;i < random_arr_length-1;i++)
    {
        seed = (a * seed + c) % m;
    	arrayseed = seed % 68 + 8;
    	random_array[i] = arrayseed;
    }
    
    return  random_array[random_arr_index]; // Generates random number between 5 and 75
}


void clear_player_right(void) // Deletes the old right side of player when it moves left
{
	const char *player_delete_part1 = "  ";
	const char *player_delete_part2 = "  ";
	const char *player_delete_part3 = "  ";
	draw_strxy(player_delete_part1,x+3,y);
	draw_strxy(player_delete_part2,x+4,y+1);
	draw_strxy(player_delete_part3,x+5,y+2);
}

void clear_player_left(void) //Deletes the old right side of player when it moves right
{
	const char *player_delete = "  ";
	draw_strxy(player_delete,x-1,y);
	draw_strxy(player_delete,x-1,y+1);
	draw_strxy(player_delete,x-1,y+2);
}

void draw_bullet(void) //Drawing bullet
{
	const char *bullet = ".";
	if(!gameover) {
	draw_strxy(bullet,x,y+2);
	}
}

void draw_player(void) //Drawing player
{
	const char *player_part1 = "  /O\\";
	const char *player_part2 = " /+|+\\";
	const char *player_part3 = "/__|__\\";
	if(!gameover) {
	draw_strxy(player_part1,x,y);
	draw_strxy(player_part2,x,y+1);
	draw_strxy(player_part3,x,y+2);
	}
}
void clear_enemy(void)
{
	const char *enemy_part1 = "     ";
	const char *enemy_part2 = "     ";
	const char *enemy_part3 = "     ";
	if(!gameover) {
	draw_strxy(enemy_part1,enemyX,enemyY);
	draw_strxy(enemy_part2,enemyX,enemyY+1);
	draw_strxy(enemy_part3,enemyX,enemyY+2);
	}
}
void clear_enemy_up(void)
{
	const char *enemy_delete = "     ";
	draw_strxy(enemy_delete,enemyX,enemyY-1);
}
void draw_enemy(void) //Drawing enemy
{
	const char *enemy_part1 = "/ | \\";
	const char *enemy_part2 = "--O--";
	const char *enemy_part3 = "\\ | /";
	if(!gameover) {
	draw_strxy(enemy_part1,enemyX,enemyY);
	draw_strxy(enemy_part2,enemyX,enemyY+1);
	draw_strxy(enemy_part3,enemyX,enemyY+2);
	}
}
void gameboard(void) //game screen
{
    int i, j, x, y;
    
    kprint_newline();
    kprint_newline();
    
    draw_enemy();
    draw_player();
    
}

void game(void) // main game loop
{
    gameboard();
    while(1)
    { 
      if(checkCollision() == 1)
      {
        gameover = 1;
      }
      if(gameover == 1)
      {
        break;
      }
      if(didStart == 1)
      {
      	if(enemyY > 25) // if enemy passes the player game will over
      	{
	gameover = 1;
	break;
      	if(random_arr_index >= 99)
	    {
	      	random_arr_length += 100;
	    }
        }
        
      draw_strxy("Score: ",1,2);
      print_integer(score,0x02);
      kprint_newline();
      kprint_newline();
      draw_strxy("Level: ",1,3);
      print_integer(level,0x02);
      
      
      enemyY = enemyY + 1;
      draw_enemy();
      clear_enemy_up();
      
      
    if (score >= 50){
    sleep(65000000);
    level = 5;
    }
    else if (score >= 30){
    sleep(70000000);
    level = 4;
    }
    else if (score >= 20){
    sleep(75000000);
    level = 3;
    }
    else if (score >= 10){
    sleep(80000000);
    level = 2;
    }
    else{
    sleep(85000000);
    level = 1;
    }
      update_bullets(); 
      
      }
  }  
  clear_screen();
}

void kmain(void)
{
    
    idt_init();
    kb_init();  
    clear_screen();
    start_screen();
    game();
    end_screen();
}
