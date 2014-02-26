---
layout: post
title: Distance between two points on a torus
category: post
---

Looking for a really handy function to calculate the distance between 2 points on a torus?

Here is a simple C# implementation:

public static double Distance(Point a, Point b, int size)
{
    int x = Math.Abs(b.X - a.X);
    int y = Math.Abs(b.Y - a.Y);

    int minX = Math.Min(x, (size - x));
    minX *= minX;

    int minY = Math.Min(y, (size - y));
    minY *= minY;

    return Math.Sqrt(minX + minY);
}