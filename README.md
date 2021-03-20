# Circle_Rasterizer
An overview on circle drawing in C/C++ and acceleration with x86-SSE

-> Please read this tutorial right after ***Triangle_Rasterizer*** tutorial for better understanding <-

# Prerequisites
- Knowledge of line-based pixel filling approach (look at *Triangle_Rasterizer* repository)
- Experinece with rasterization and concept on blit
- Good knowledge of x86 assembly and C programming languages

# Optimization
- SIMD (x86-SSE) optimization 

# Let's get started.
## Line-based pixel drawing in C
In our triangle_Rasterizer, I explained how to start with the top point and approach the side point of a triangle, then for every single step, we draw a horizontal line between two limit point on triangle's borders. For a circle, the strategy is similar. I start with the toppest point on a circle (north pole or the point at &theta; = 90°) and approach the point at the leftmost part of the circle (at &theta; = 180°). For every single step, I draw a horizontal line in order to cover a semi circle from &theta; = 0° to &theta = 180°. I, then, repeat the same for angles between 180° to 360°.

## Circles

Okay, now it is time to define a circle as follows:

	typedef struct _circle {
 		point*   center;
		float    radius;
		COLORREF color;
	} circle;

I pass the circle's object to function ***line_based_dword_fillup_circle***.

	void line_based_dword_fillup_circle(circle* cr)
	{
		COLORREF* framebuffer = (COLORREF*)fb;
	
		/* strat from top point at angle 90° towards point at angle 180° */
		float        top_x              = cr->center->x;
		float        top_y              = cr->center->y - cr->radius;
		float        r_squared          = (cr->radius * cr->radius);
		float        left_side_x        = top_x - cr->radius;
		float        left_side_y        = cr->center->y;
		float        next_y             = top_y;
		float        next_x             = top_x;
		float        delY               = 0.0f;
		unsigned int this_pixel         = 0;
		unsigned int pixel_number       = 0;
		unsigned int mirror_pixel       = 0;
		unsigned int uint_center_y_copy = (unsigned int)(cr->center->y);
	
		while (next_y <= left_side_y)
		{
			delY = next_y - cr->center->y;
			next_x = top_x - sqrt(r_squared - (delY * delY));
	
			/* make an alias for better performance in next two lines */
			unsigned int next_x_copy = (unsigned int)next_x;
	
			this_pixel   = (next_x_copy + ((unsigned int)next_y * WIDTH));
			mirror_pixel = (next_x_copy + WIDTH * (uint_center_y_copy + (unsigned int)(cr->center->y - next_y)));
			pixel_number = (unsigned int)((cr->center->x - next_x)) << 1;
			while (pixel_number--)
				framebuffer[this_pixel++] = framebuffer[mirror_pixel++] = cr->color;
			next_y++;
		}
	}

While we go one step in y direction down, we calculate the x component of the left side limit pixel. Based on this x component, we therefore calcualte the x component of the limit pixel at the oppsite side of the circle. Then fill up this scanline.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/Circle_Rasterizer/blob/main/images/filled_in_c.PNG"/>
</p>

Similar words to the triangle blit, the heart of the rasterizer is where we can work on to accelerate the blitting. For this reason I use x86-SSE optimization. If we concentrate on the way both limit points defined, we come up with this fact that, it cannot use packed floating point instructions since a line pixel can be anywhere on screen and any attampt to make it 16-byte aligned, would be a speed killer, and better to stick with the un-aligned instructions. Also I used inline assembly and not a separate .asm file (to tolerate the overhead of function epilogue and prologue).

	void accelerated_line_based_xmmword_fillup_circle(circle* cr)
	{
		/* construct super color */
		__declspec(align(16)) COLORREF _super_color[4] = { cr->color, cr->color , cr->color , cr->color };
	
		/* strat from top point at angle 90° towards point at angle 180° */
		float        top_x              = cr->center->x;
		float        top_y              = cr->center->y - cr->radius;
		float        r_squared          = (cr->radius * cr->radius);
		float        left_side_x        = top_x - cr->radius;
		float        left_side_y        = cr->center->y;
		float        next_y             = top_y;
		float        next_x             = top_x;
		float        delY               = 0.0f;
		unsigned int this_pixel         = 0;
		unsigned int pixel_number       = 0;
		unsigned int mirror_pixel       = 0;
		unsigned int uint_center_y_copy = (unsigned int)(cr->center->y);
	
		while (next_y <= left_side_y)
		{
			delY = next_y - cr->center->y;
			next_x = top_x - sqrt(r_squared - (delY * delY));
	
			/* make an alias for better performance in next two lines */
			unsigned int next_x_copy = (unsigned int)next_x;
	
			this_pixel   = (next_x_copy + ((unsigned int)next_y * WIDTH));
			mirror_pixel = (next_x_copy + WIDTH * (uint_center_y_copy + (unsigned int)(cr->center->y - next_y)));
			pixel_number = (unsigned int)((cr->center->x - next_x)) << 1;
	
			//--------------------------- SIMD OPTIMIZED LINE RASTERIZER -------------------------------
			/* 16 byte */
			unsigned int   total_bytes_in_line = pixel_number << 2;
			unsigned int   xmmwords_number = total_bytes_in_line >> 4;
			unsigned int   _16_bytes_blocks_in_line = (xmmwords_number << 4);
			unsigned char* start_of_line_in_framebuffer = fb + (this_pixel << 2);
			unsigned char* start_of_mirror_line_in_framebuffer = fb + (mirror_pixel << 2);
			unsigned int*  start_of_final_dwords_in_framebuffer = NULL;
			unsigned int*  start_of_final_mirror_dwords_in_framebuffer = NULL;
			__asm {
				mov    edi, start_of_line_in_framebuffer
				mov    esi, start_of_mirror_line_in_framebuffer
				mov    ecx, xmmwords_number
			// sanity check
				cmp    ecx, 0
				je     _accelerated_blit_end_
				movaps  xmm0, _super_color
				movaps  xmm1, xmm0
			_accelerated_blit_:
				movups xmmword ptr[edi], xmm0
				movups xmmword ptr[esi], xmm1
				sub    ecx, 1
				add    edi, 16
				add    esi, 16
				cmp    ecx, 0
				je     _accelerated_blit_end_
				jmp    _accelerated_blit_
			_accelerated_blit_end_:
				mov    start_of_final_dwords_in_framebuffer, edi
				mov    start_of_final_mirror_dwords_in_framebuffer, esi
			}
	
			/* 4 byte */
			unsigned int dwords = (total_bytes_in_line - _16_bytes_blocks_in_line) >> 2;
			while (dwords--)
				*start_of_final_dwords_in_framebuffer++ = *start_of_final_mirror_dwords_in_framebuffer++ = cr->color;
			//------------------------------------------------------------------------------------------
	
			next_y++;
		}
	}

This is, in my opinion, the fastest way to paint a aliased circle based on the scanline method with SIMD acceleration.

In function ***accelerated_line_based_xmmword_fillup_circle***, all the calculations are similar to those of ***line_based_dword_fillup_circle***, but instead of DWORD blit, I separated blocks of XMMWORDs and used 16-byte blit and finally some DWORD blit at the very end. 

The benchmarking made it clear that ***accelerated_line_based_xmmword_fillup_circle*** is simply 3.8 times faster than ***line_based_dword_fillup_circle***. A single core CPU spent nearly ***0.68 ms*** to paint the represented triangle above with ***line_based_dword_fillup_circle*** and the same with ***0.19 ms*** in ***accelerated_line_based_xmmword_fillup_circle***.

## Some notes on hardware accelerated circle blit

Imagine a GPU designed for circle raterization. This GPU has over 1000 parallel cores. In our *accelerated_line_based_xmmword_fillup_circle*, there is 

	while (next_y <= left_side_y)
	{
		...
		next_y++;
	}
	
Our GPU can break this statement into **dy** cores (if dy < number_of_GPU_cores) and do all calculations in parallel, or into **number_of_GPU_cores** cores (if number_of_GPU_cores < dy) and in some small steps do the whole painting. With this in mind, we see that if dy < number_of_GPU_cores we get dy factor in acceleration and if number_of_GPU_cores < dy, we would get (number_of_GPU_cores * (dy / number_of_GPU_cores)) + (dy % number_of_GPU_cores) factor in acceleration. 

I mentioned at the end of my ***Triangle_Rasterizer*** tutorial that one can draw circles from triangles building blocks and spend nearly 300 msec to draw a large circle. As I also mentioned there, this is not efficient at all. Here I tried to show you that we can draw such a large circle with time below 200 &mu;sec.

# Anti aliasing
