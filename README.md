# Circle_Rasterizer
An overview on circle drawing in C/C++ and acceleration with x86-SSE.

Please read this tutorial right after ***Triangle_Rasterizer*** tutorial for better understanding.

# Prerequisites
- Knowledge of line-based pixel filling approach (look at *Triangle_Rasterizer* repository)
- Experinece with rasterization and concept on blit
- Good knowledge of x86 assembly and C programming languages

# Optimization
- SIMD (x86-SSE) optimization 

# Let's get started.
## Line-based pixel drawing in C
In our triangle_Rasterizer, I explained how to start with the top point and approach the side point of a triangle, then for every single step, we draw a horizontal line between two limit point on triangle's borders. For a circle, the strategy is similar. I start with the toppest point on a circle (north pole or the point at tt = 90°) and approach the point at the leftmost (at &theta = 180°) part of the circle. For every single step, I draw a horizontal line in order to cover a semi circle from &theta = 0° to &theta = 180°. I then repeat the same fot angles between 180° to 360°.

## Circles

Okay, now it is ime to define a circle as follows:

    typedef struct _circle {
      point*   center;
		  float    radius;
		  COLORREF color;
    } circle;




Then I start from top point and steps through the line between this point and side_point_1. I go one pixel down, then calculate the coressponding pixel between top point and side_point_2. I have now two limit points on a scanline and all pixels in between have to be painted. I have a function called ***line_based_dword_fillup_triangle*** for this reason. It is insanely fast and efficient.

	void line_based_dword_fillup_triangle(triangle* tr)
	{
		COLORREF* framebuffer = (COLORREF*)fb;
	
		/* sort points of triangle */
		point* top_point    = NULL;
		point* side_point_1 = NULL;
		point* side_point_2 = NULL;
		FIND_TOP_POINT((tr), top_point, side_point_1, side_point_2);
	
		/* strat from top point towards side point 1 */
		float x0 = (float)(top_point->x);
		float y0 = (float)(top_point->y);
		float x  = (float)(side_point_1->x);
		float y  = (float)(side_point_1->y);
		float dx = x - x0;
		float dy = y - y0;
		float m  = dy / dx;
		float mprime = 1.0 / m;
		float opposite_m = ((float)(side_point_2->y) - y0) / ((float)(side_point_2->x) - x0);
		float gamma = (m / opposite_m);
		float next_y = y0;
		float next_x = x0;
		float opposite_x = 0;
		unsigned int this_pixel;
		unsigned int pixel_number;
		
		// the heart of the rasterizer
		while (next_y < y)
		{		
			next_x = x0 + ((next_y - y0) * mprime);
			opposite_x = (gamma * (next_x - x0)) + x0;
			this_pixel = ((unsigned int)next_x + ((unsigned int)next_y * WIDTH));
			pixel_number = ((unsigned int)opposite_x - (unsigned int)next_x);
			while (pixel_number--)
			{
				framebuffer[this_pixel] = tr->color;
				this_pixel++;
			}
			next_y++;
		}
	}

There is room for some minor optimization that I leave for readers to play with, but this function is the best to use scanline method for painting the triangle.

While we go one step in y direction down, we calculate the slope of the line between top_point and side_point_1 and thus the x component of the left side limit pixel. Based on this x component, we therefore calcualte the x component of the limit pixel falling on the line between top_point and side_point_2. Then fillup this scanline.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/Triangle_Rasterizer/blob/main/images/filled_in_c.PNG"/>
</p>

The heart of the rasterizer is where we can work on to accelerate the blitting. For this reason I use x86-SSE optimization. If we concentrate on the way both limit points defined, we come up with this fact that, it cannot use packed floating point instructions since a line pixel can be anywhere on screen and any attampt to make it 16-byte aligned, would be a speed killer, and better to stick with the un-aligned instructions. Also I used inline assembly and not a separate .asm file (to tolerate the overhead of function epilogue and prologue).

	void accelerated_line_based_xmmword_fillup_triangle(triangle* tr)
	{
		/* sort points of triangle */
		point* top_point    = NULL;
		point* side_point_1 = NULL;
		point* side_point_2 = NULL;
		FIND_TOP_POINT((tr), top_point, side_point_1, side_point_2);
	
		/* construct super color */
		__declspec(align(16)) COLORREF _super_color[4] = { tr->color, tr->color , tr->color , tr->color };
	
		/* strat from top point towards side point 1 */
		float x0         = (float)(top_point->x);
		float y0         = (float)(top_point->y);
		float x          = (float)(side_point_1->x);
		float y          = (float)(side_point_1->y);
		float dx         = x - x0;
		float dy         = y - y0;
		float m          = dy / dx;
		float mprime     = 1.0 / m;
		float opposite_m = ((float)(side_point_2->x) - x0) / ((float)(side_point_2->y) - y0);
		float gamma      = (m * opposite_m);
		float next_y     = y0;
		float next_x     = x0;
		float opposite_x = 0;
		unsigned int this_pixel;
		unsigned int pixel_number;
	
		while (next_y < y)
		{
			next_x       = x0 + ((next_y - y0) * mprime);
			opposite_x   = (gamma * (next_x - x0)) + x0;
			this_pixel   = ((unsigned int)next_x + ((unsigned int)next_y * WIDTH));
			pixel_number = ((unsigned int)opposite_x - (unsigned int)next_x);
	
	
			//--------------------------- SIMD OPTIMIZED LINE RASTERIZER -------------------------------
			/* 16 byte */
			unsigned int   total_bytes_in_line                  = pixel_number << 2;
			unsigned int   xmmwords_number                      = total_bytes_in_line >> 4;
			unsigned int   _16_bytes_blocks_in_line             = (xmmwords_number << 4);
			unsigned char* start_of_line_in_framebuffer         = fb + (this_pixel << 2);
			unsigned int*  start_of_final_dwords_in_framebuffer = NULL;
			__asm {
					mov    edi, start_of_line_in_framebuffer
					mov    ecx, xmmwords_number
				// sanity check
					cmp    ecx, 0
					je     _accelerated_blit_end_
					movaps  xmm0, _super_color
				_accelerated_blit_:
					movups xmmword ptr[edi], xmm0
					sub    ecx, 1
					add    edi, 16
					cmp    ecx, 0
					je     _accelerated_blit_end_
					jmp    _accelerated_blit_
				_accelerated_blit_end_:
					mov    start_of_final_dwords_in_framebuffer, edi
			}
			
			/* 4 byte */
			unsigned int dwords = (total_bytes_in_line - _16_bytes_blocks_in_line) >> 2;
			while (dwords--)
				*start_of_final_dwords_in_framebuffer++ = tr->color;
			//------------------------------------------------------------------------------------------
	
			next_y++;
		}
	}

This is, in my opinion, the fastest way to paint a triangle based on the scanline method with SIMD acceleration. Of course some very little optimizations can be done, but they are left for readers to play with.

In function ***accelerated_line_based_xmmword_fillup_triangle***, all the calculations are similar to those of ***line_based_dword_fillup_triangle***, but instead of DWORD blit, I separated blocks of XMMWORDs and used 16-byte blit and finally some DWORD blit at the very end. 

The benchmarking made it clear that ***accelerated_line_based_xmmword_fillup_triangle*** is simply 5.5 times faster than ***line_based_dword_fillup_triangle***. A single core CPU spent nearly ***0.43 ms*** to paint the represented triangle above with ***line_based_dword_fillup_triangle*** and the same with ***0.08 ms*** in ***accelerated_line_based_xmmword_fillup_triangle***.

## Some notes on hardware accelerated triangle blit

Imagine a GPU designed for triangle raterization. This GPU has over 1000 parallel cores. In our *accelerated_line_based_xmmword_fillup_triangle*, there is 

	while (next_y < y)
	{
		...
		next_y++;
	}
	
Our GPU can break this statement into **dy** cores (if dy < number_of_GPU_cores) and do all calculations in parallel, or into **number_of_GPU_cores** cores (if number_of_GPU_cores < dy) and in some small steps do the whole painting. With this in mind, we see that if dy < number_of_GPU_cores we get dy factor in acceleration and if number_of_GPU_cores < dy, we would get (number_of_GPU_cores * (dy / number_of_GPU_cores)) + (dy % number_of_GPU_cores) factor in acceleration. 

If our imaginated GPU draws circles and ellipses also based on triangle building blocks, one can see that for a large circle with radius of 400 (something like the hypotenuse of triangle above), if we start from 0° to 360° with steps 0.1°, we will be ended up to 3600 * 0.08 ms = 288 ms. Of course this is not the way to draw a circle, but just to give you a flavour of what would be the result of designing a game with everything based on triangles (please look at my ***Circle_Rastrizer*** tutorial for a comprehensive approach).
		
