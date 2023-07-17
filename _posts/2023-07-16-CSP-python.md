---
title: Mastering Material Usage Optimization For Your Projects - A Python-Powered Solution
date: 2023-07-16
---

>This article was originally published on July 15, 2023 on [linkedin.com](https://www.linkedin.com/pulse/mastering-material-usage-optimization-your-projects-ilgiz-nigmatullin)

<style>
  .trimmed-cover {
  object-fit: cover;
  width: 100%;
  height: 384px;
  overflow: hidden;
  object-position: center 40%;
}
</style>

<div class="trimmed-cover">
  <img src="https://media.licdn.com/dms/image/D4E12AQGRiLb98y0mIg/article-cover_image-shrink_720_1280/0/1689256150839?e=1695254400&v=beta&t=_tRuoeDDpoDlIMHphobFEC7xoXJxOXC8HLo3uwsy6r0" alt="Tetris">
</div>

<p align="center">
  <em>Tetris Vectors by Vecteezy: https://www.vecteezy.com/free-vector/tetris</em>
</p>

### Intro

In the previous [post](https://inigmat.github.io/home/2023/06/27/Cutting_calcs.html), I shared links to some ready-to-use online calculators that provide a quick tool for finding the optimal amount of materials to order for your project. However, what if you want to add additional features to these calculators or work around limitations like the number of parts? In this article, I will show you how to write Python code to solve this problem and create your own flexible solution.

To illustrate the process, we will use the example of a bar schedule for concrete foundations, where we need to cut rebar with specific lengths and quantities.

<p align="center">
  <img src="https://media.licdn.com/dms/image/D4E12AQHkIKi1cCBszg/article-inline_image-shrink_1500_2232/0/1689400156113?e=1695254400&v=beta&t=goz9fIxjZbQBzIgsvGnK0ojfAdtdflAJIbQTquwJmKQ" alt="Картинка">
</p>

<p align="center">
  <em>We use some sample data</em>
</p>

### Part 1. Quick Start. Optimizing the required number of rebar 

Let's start by calculating the number of rods to order based on the demand indicated in the table. We can convert the table into a list of required rebar parts, repeating each element as many times as indicated in the table. In this case, we have a list of 25 elements.

```python
parts = [2350, 2350, 2350, 2350, 2800, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 3030, 4230, 4230, 4230, 4230, 4230, 4230]
```

Next, we can use the ['binpacking](https://pypi.org/project/binpacking/#description)['](https://pypi.org/project/binpacking/#description)  package to solve the problem in just one line of code:

```python
rods = binpacking.to_constant_volume(parts, STOCK_REBAR_LENGTH)
```

By adding print functions to display the results, we can see the required number of bars and the cutting scheme:

```python
import binpacking

STOCK_REBAR_LENGTH = 11700

rods = binpacking.to_constant_volume(parts, STOCK_REBAR_LENGTH)
print(f'Required number of bars: {len(rods)} pcs')
print(f'Cutting scheme: {rods}')
```

The output will show the required number of bars and the cutting scheme for the rebar:

```python
Required number of bars: 7 pcs
Cutting scheme: [[4230, 4230, 3030], [4230, 4230, 3030], [4230, 4230, 3030], [3030, 3030, 3030, 2350], [3030, 3030, 3030, 2350], [3030, 3030, 3030, 2350], [3030, 3030, 2800, 2350]]
```

Although the blade thickness is not taken into account (which can be neglected in our case) and the variance of rebar in stock cannot be defined (as we assume only one length of 11700 mm), we have managed to find the optimal amount of rebar to order and determine the cutting scheme with just a few lines of code..

### Part 2. Optimization of the required quantity of reinforcement for the entire structure

Now that we have seen how this solution works, let's apply it to larger structures with different diameters of rebar. 

We have prepared a table with the demand from the design documentation:

<p align="center">
  <img src="https://media.licdn.com/dms/image/D4E12AQFCFa6Av_mGsQ/article-inline_image-shrink_1500_2232/0/1689400233647?e=1695254400&v=beta&t=7WVESex47NKbleTWULLc_zeByMowdVDOMGZnhQfoJXg" alt="Картинка">
</p>

To see the full code and results, you can refer to the Jupyter notebook available at the following link: <https://nbviewer.org/github/inigmat/exupery/blob/main/CSP_binpacking.ipynb>

After optimization, we obtained the following results for the required number of reinforcement bars and the total scrap percentage for each diameter:

```
Required number of reinforcement bars with diameter 10 mm: 168 pcs
Total scrap: 2.32%
Required number of reinforcement bars with diameter 12 mm: 138 pcs.
Total scrap: 1.47%
Required number of reinforcement bars with diameter 16 mm: 143 pcs.
Total scrap: 3.76%
Required number of reinforcement bars with diameter 20 mm: 759 pcs.
Total scrap: 4.82%.
```

We also generated a chart showing the cutting plan for the bars.

<p align="center">
  <a href="https://docs.google.com/spreadsheets/d/e/2PACX-1vTj-Fr6BGuoJlJdEglvPbYqOPi1k0Gwjb4lcTJ2aR8upi6esjDX2p8qej6nsMFF2NqCcjJrPBzsPSTQ/pubhtml?gid=1092469187&single=true">
    <img src="https://media.licdn.com/dms/image/D4E12AQHw04l3XegRkg/article-inline_image-shrink_1000_1488/0/1689402028786?e=1695254400&v=beta&t=BKaud21xLZyBllcGnR9JzNjJztdF-qCxmEqxR5LfJ-s" alt="Картинка">
  </a>
</p>

<p align="center">
  <em>For a more detailed view of the data: <a href="https://docs.google.com/spreadsheets/d/1pVxdnsfdEvEntjyNfpz_MTDin3x5JktN83TTT8Kvy3U/edit?usp=sharing">link</a> 
  </em>
</p>

### Part 3. Looking under the hood

While optimizing the cutting of items from larger pieces to satisfy the demand and minimize waste, we are actually solving the [Cutting Stock Problem](https://en.wikipedia.org/wiki/Cutting_stock_problem) (CSP). This problem involves finding the most efficient way to cut larger pieces into smaller items, considering constraints such as item sizes and quantities. CSP is a variant of the [packing problem](https://developers.google.com/optimization/pack/bin_packing?hl=en), specifically [bin packing problem](https://en.wikipedia.org/wiki/Bin_packing_problem), which aims to find the fewest bins that can hold all the items. This is why we use the 'binpacking' package, as it is designed to address this problem.

The main feature of the ['binpacking'](https://pypi.org/project/binpacking/#description)  package is the [first-fit decreasing algorithm](https://en.wikipedia.org/wiki/First-fit-decreasing_bin_packing), which is much faster than solutions based on [mixed-integer programming](https://developers.google.com/optimization/pack/bin_packing?hl=en) when dealing with a large number of elements.

More read here:

-   [Bin packing and cutting stock problems](https://scipbook.readthedocs.io/en/latest/bpp.html)
-   [Exploring the Bin Packing Problem](https://medium.com/swlh/exploring-the-bin-packing-problem-f54a93ebdbe5)

### Conclusion

By implementing the code presented in this article, we can find the optimal quantity of materials to order for a given project. The approach provides flexibility and additional features compared to online calculators. With the help of the 'binpacking' package, we can efficiently solve the Cutting Stock Problem.

Whether you are working on a small-scale project or a larger construction endeavor, this method can help save time, reduce waste, and ensure efficient material usage.
