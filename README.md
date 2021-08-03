# VEX particles
This is an example for a simple particle system written in VEX. This sample uses some fBM code by [Inigo Quilez](https://www.iquilezles.org/www/articles/warp/warp.htm) as well as an additive particle shading technique by [Simon Fiedler](https://simonfiedler.de/) featured on [Entagma](https://entagma.com/).

## Initialize the particles

Set initial values for life, decay and velocity attributes in a point wrangle
```C
f@life = (0.2f + rand(@ptnum + 123) * 0.8f);
f@origlife = f@life;
f@decay = 0.3f + rand(@ptnum + 753) * 0.7f;

float velmax = .02f;
float velmin = velmax * 0.5f;
v@v = set(
    (0.2f + rand(@ptnum + 9651) * 0.8f) * velmax - velmin,
    (0.2f + rand(@ptnum + 1593) * 0.8f) * velmax - velmin,
    (0.2f + rand(@ptnum + 3574) * 0.8f) * velmax - velmin
);
```

## Particles update
Add a point wrangle in a SOP Solver to compute particle position and velocity each frame.
```C
// https://www.iquilezles.org/www/articles/fbm/fbm.htm
float   fbm( const vector x )
{
    int NUM_NOISE_OCTAVES = 9;
    vector fx = x;
    float v = 0.0;
    float a = 0.5;
    vector shift = set(100.0f);
    
    for (int i = 0; i < NUM_NOISE_OCTAVES; ++i)
    {
        v += a * noise(fx);
        fx = fx * 2.0f + shift;
        a *= 0.5f;
    }
    return v;
}

// https://www.iquilezles.org/www/articles/functions/functions.htm
float   parabola( float x; float k )
{
    return pow(4.0f * x * (1.0f - x), k);
}

// parameters ----------------------------------------------------------------
float uCurlSpeed = chf("curl_speed");
float uCurlRoughness = chf("curl_roughness");
float uCurlScale = chf("curl_scale");
float uVelDamping = chf("vel_damping");
float t = @Time * chf("time_scale");
float TWO_PI = 2.0f * M_PI;
float uDelta = 0.016f;

//-----------------------------------------------------------------------------
float x = fbm(@P.xyz - sin(t) + fbm(@P.xyz + t));
float y = fbm(@P.zxy - cos(t) + fbm(@P.zxy + t));
float z = fbm(@P.yzx - sin(t) + fbm(@P.yzx + t));

x = fbm(x) * 2.0f - 1.0f;
y = fbm(y) * 2.0f - 1.0f;
z = fbm(z) * 2.0f - 1.0f;

float tx = @Time * 0.01f;
float ty = @Time * 0.02f;
float tz = @Time * 0.04f;

x += cos(tx + (fbm(x + v@v.x) * 2.0f - 1.0f) * TWO_PI);
y += cos(ty + (fbm(y + v@v.y) * 2.0f - 1.0f) * TWO_PI);
z += cos(tz + (fbm(z + v@v.z) * 2.0f - 1.0f) * TWO_PI);
//-----------------------------------------------------------------------------

//-----------------------------------------------------------------------------
float max = chf("noise_scale");
float min = max * 0.5f;
vector sp = @P * 1.0f;

float x0 = fbm(sp.xyz + cos(t) + fbm(v@v.xyz + fbm(sp.xyz + sin(t)))) * max - min;
float y0 = fbm(sp.zxy + sin(@Time) + fbm(v@v.zxy + fbm(sp.zxy + cos(t)))) * max - min;
float z0 = fbm(sp.yzx + cos(t) + fbm(v@v.yzx + fbm(sp.yzx + sin(@Time)))) * max - min;

x0 -= cos(TWO_PI * x0) * uDelta;
y0 -= cos(TWO_PI * y0) * uDelta;
z0 -= cos(TWO_PI * z0) * uDelta;
//-----------------------------------------------------------------------------

vector pa = set(x, y, z);
vector pb = set(x0, y0, z0);
vector va = (uDelta * uCurlSpeed * curlnoise((@P + pa) * uCurlRoughness - @Time) * uCurlScale * 0.25f);
vector vb = uDelta * uCurlSpeed * (pb * uCurlRoughness) * uCurlScale;

@P += v@v;
v@v += lerp(va, vb, (1.0f + cos(@Time)) * 0.5f);
v@v *= uVelDamping;

f@life -= f@decay * uDelta;

float max2 = 1.6f;
float min2 = max2 * 0.5f;

if (f@life <= 0.0f)
{
    @P.x = rand(@ptnum + @P.x + @Time) * max2 - min2;
    @P.y = rand(@ptnum + @P.y + @Time) * max2 - min2;
    @P.z = rand(@ptnum + @P.z + @Time) * max2 - min2;
    v@v *= 0.2f;
    f@life = (0.2f + rand(@ptnum + @Time + @Frame) * 0.8f);
    f@origlife = f@life;
}

@pscale = parabola(f@life / f@origlife, 4.0f);
```
