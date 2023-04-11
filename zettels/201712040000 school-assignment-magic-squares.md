
---
title: "School Assignment: Magic Squares"
slug: "school-assignment-magic-squares"
date: "2017-12-04"
summary: ""
---

#software-development #dotnet

On keeping up with the regular bullshit at school; I get to skip a part off my minor. Anyhow, this one is on doing 'algorithms and data structures'.

Recently someone finally gave a decent explanation about arrays. That was about time after three years of computer science classes. Great. Even better, we get to use those now, by generating magic squares.

![http://localhost:3000/uploads/magicsquareexample.svg](/uploads/magicsquareexample_a7552b0fd2.svg)

Magic squares are great. The goal with these things is that when summed up, every row, column or diagonal has the same value as shown in the example above. There's [an algorithm for generating these things](https://en.wikipedia.org/wiki/Magic_square#Types_of_construction). I've implemented a sloppy algorithm which can generate these things as long as the row and column size are an odd number;

```csharp
int num = 5;
int[,] matrix = new int[num, num];

var x = ((num) / 2);
var y = num-1;

int i = 1;

int upper = num\*num;

var prev_x = x;
var prev_y = y;

while (i <= upper)
{
    if (matrix[x, y] == 0) {
        matrix[x, y] = i;
        i++;

        prev_x = x;
        prev_y = y;

        x = x < num - 1 ? x + 1 : 0;
        y = y < num - 1 ? y + 1 : 0;
    } else {
        x = prev_x;
        y = prev_y - 1;
    }
}

matrix.Dump();
```

*I can definitely recommend LinqPad for playing with fiddles like this as it will give a clear view of what you are doing.*

![http://localhost:3000/uploads/versions/screen-shot-2017-12-04-at-11-29-44-pm---x----2344-1886x---.png](/uploads/screen_shot_2017_12_04_at_11_29_44_pm_x_2344_1886x_16afa7f36d.png)

## Visualizing Squares

While we can generate magic squares now there's bonus points for visualizing these. Hmm. How does one go, without having to bundle LinqPad. There's like two options here;

1. You do bundle LinqPad
2. You start generating grids in a console app
3. You think of something else

Originality for the win! Actually this is the part which took more time then programming the algorithm itself.

So the go to solution was to generate bitmaps of these grids, each pixel's color representing a value on the grid. This one was actually combined with stress testing the algorithm to see if I could generate a 24 bit grid or more, which succeeded by the way. There was no reason to go higher though, as there are no more then like 16 million colors available.

The trick was in converting a value between 0 and 2^24 to a RGB color value. See the example below;

```csharp
using (var image = new Bitmap(num, num)) {
    var multiplier = 16777216 / upper;

    for (var i = 0; i < num; i++) {
        for (var ii = 0; ii < num; ii++) {
            var hex = (matrix[i, ii] \* multiplier).ToString("X8");

            var color = Color.FromArgb(
                (byte)255,
                (byte)(Convert.ToUInt32(hex.Substring(6, 2), 16)),
                (byte)(Convert.ToUInt32(hex.Substring(2, 2), 16)),
                (byte)(Convert.ToUInt32(hex.Substring(4, 2), 16))
            );

            image.SetPixel(i, ii, color);
        }
    }

    image.Save($"./{Guid.NewGuid().ToString()}.bmp");
}
```

The results, when playing with color codings, can be quite astonishing, epileptic attack inducing and trippy at the same time:

![http://localhost:3000/uploads/versions/2e6d6b17-39ac-4152-95e8-5d9824132220---x----4095-4095x---.bmp](/uploads/2e6d6b17_39ac_4152_95e8_5d9824132220_x_4095_4095x_574c5ed8a2.bmp)

Thankfully there is no need to copy paste this code as it's [all available on GitHub](https://github.com/CorstianBoerman/SquaryMcSquareFace).
