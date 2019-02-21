$$ 显然OA = OA' $$
$$ 而OA = \frac{y_1}{sin(a)} = \frac{x_1}{cos(a)} $$
$$ 且OA' = \frac{y_2}{sin(a + b)} = \frac{x_2}{cos(a + b)}$$
$$ 令它们都等于r$$
$$则 y_2 = r * (sin(a) * cos(b) + cos(a) * sin(b)) = y_1 * cos(b) + x_1 * sin(b)$$
$$则 x_2 = r * (cos(a) * cos(b) - sina(a) * sin(b)) = x_1 * cos(b) - y_1 * sin(b)$$
---
$$
        \begin{matrix}
        scaleX|cos(rotateZ) & tan(skewX)|-sin(rotateZ) & translateX \\
        tan(skewY)|sin(rotateZ) & scaleY|cos(rotateZ) &  translateY\\ 
        0 & 0 & 1 \\
        \end{matrix} 
        \,\,\,\, *\,\,\,
        \begin{matrix}
        x \\
        y \\ 
        1 \\
        \end{matrix} 
$$