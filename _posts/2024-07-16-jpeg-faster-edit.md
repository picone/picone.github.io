---
layout: post
title: 更快速的 JPEG 图片剪裁及缩放
description: 
---

[这篇文章]({% post_url 2024-07-14-jpeg-encode %})介绍了 JPEG 编码的过程，我们在使用的过程中其实有更高级的功能可以大幅减少计算量，
提高图片的处理速度。这篇文章将针对 JPEG 图片的剪裁和缩放进行讨论。

# 图片剪裁

我们知道，JPEG 编码时候将图片按照等规格的 MCU 来存放，MCU 是指 8x8 或者 16x16 大小的图片块，所有图片块加起来组成整个图片。此时聪明的你可能
已经想到，可以根据剪裁的需要解码必要的 MCU，其余 MCU 跳过即可。

这个想法可以使用 libjpeg 底层的 API 来实现，参考文档 [Partial image decompression](https://github.com/libjpeg-turbo/libjpeg-turbo/blob/0566d51e092f8c75e868d54ffa3670ce0949a46e/libjpeg.txt#L807-L893)。

```c
img_decoded = (JSAMPROW)malloc(sizeof(JSAMPLE) * crop_width * crop_height * dinfo.num_components);

real_left = (JDIMENSION)crop_left;
real_width = (JDIMENSION)crop_width;

// 需要局部解码图片的话，使用 real_left 和 real_width，因为解码必须整个MCU操作，最终的出来的行还需要一次拷贝才完整。
// 调用 jpeg_crop_scanline 这个函数是为了保证实际解码的部分是完整的MCU。这时候 real_left 和 real_width 会被修改。
if (options->crop.left > 0 || crop_width < dinfo.image_width) {
    jpeg_crop_scanline(&dinfo, &real_left, &real_width);
}

// 横向跳过指定行数
if (crop_top > 0) {
    jpeg_skip_scanlines(&dinfo, (JDIMENSION)crop_top); // 实际上应该校验返回值是否真的都能成功跳过了。
}

// 计算解码每行需要的字节数和实际提取出来有效的字节数
img_row_size = sizeof(JSAMPLE) * real_width * dinfo.num_components;
img_real_row_size = sizeof(JSAMPLE) * crop_width * dinfo.num_components;

// 申请内存，用于接收 scanline 的数据，即比实际大的数据
img_row = (JSAMPROW)malloc(img_row_size);

// real-left 是指实际解码的像素 left 坐标，很可能都在 crop_left 的左面，img_row 接收到的数据会比 crop_left 多，要自己剪裁下忽略掉，
// 这里计算需要忽略的字节数。
img_real_row_start = img_row + (sizeof(JSAMPLE) * (crop_left - real_left) * dinfo.num_components);

current_row = img_decoded; // 指向当前第一行的指针
while (dinfo.output_scanline < crop_top + crop_height) {
    // 逐行读取 scanlines，每行结果用 img_row 来接，因为 MCU 只能整个解码，实际 real_width 有可能比 crop_width 大。
    // 每次只读一行，每行的前面有 (crop_left - real_left) 个像素被剪裁了。
    jpeg_read_scanlines(&dinfo, &img_row, 1);
    
    // 实际上读出来的 scanline 会多于需要的像素，所以复制一下到 img_decoded
    memcpy(current_row, img_real_row_start, img_real_row_size);

    // 每读完一行，current_row 就移动到下一行，方便后续继续追加数据
    current_row += img_row_size;
}
```
