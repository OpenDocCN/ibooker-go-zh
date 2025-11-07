## 附录. 解答

本附录提供了课程结束练习和核心项目的解答。请注意，任何问题都有不止一个解决方案。


##### 备注

您可以从 Manning 网站 [www.manning.com/books/get-programming-with-go](http://www.manning.com/books/get-programming-with-go) 下载这些解答和其余的源代码，或者在网上浏览 [github.com/nathany/get-programming-with-go](http://github.com/nathany/get-programming-with-go) 的源代码。


### 单元 0

#### 第 1 课

##### 实验: playground.go

```
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello, Nathan")
    fmt.Println(" hola")
}
```

### 单元 1

#### 第 2 课

##### 实验: malacandra.go

```
package main

import "fmt"

func main() {
    const hoursPerDay = 24

    var days = 28
    var distance = 56000000 // km

    fmt.Println(distance/(days*hoursPerDay), "km/h")
}
```

#### 第 3 课

##### 实验: guess.go

```
package main

import (
    "fmt"
    "math/rand"
)

func main() {
    var number = 42

    for {
        var n = rand.Intn(100) + 1
        if n < number {
            fmt.Printf("%v is too small.\n", n)
        } else if n > number {
            fmt.Printf("%v is too big.\n", n)
        } else {
            fmt.Printf("You got it! %v\n", n)
            break
        }
    }
}
```

#### 第 4 课

##### 实验: random-dates.go

```
package main

import (
    "fmt"
    "math/rand"
)

var era = "AD"

func main() {
    for count := 0; count < 10; count++ {
        year := 2018 + rand.Intn(10)
        leap := year%400 == 0 || (year%4 == 0 && year%100 != 0)
        month := rand.Intn(12) + 1

        daysInMonth := 31
        switch month {
        case 2:
            daysInMonth = 28
            if leap {
                daysInMonth = 29
            }
        case 4, 6, 9, 11:
            daysInMonth = 30
        }

        day := rand.Intn(daysInMonth) + 1
        fmt.Println(era, year, month, day)
    }
}
```

#### 核心项目 5

##### 实验: tickets.go

```
package main

import (
    "fmt"
    "math/rand"
)

const secondsPerDay = 86400

func main() {
    distance := 62100000
    company := ""
    trip := ""

    fmt.Println("Spaceline        Days Trip type  Price")
    fmt.Println("======================================")

    for count := 0; count < 10; count++ {
        switch rand.Intn(3) {
        case 0:
            company = "Space Adventures"
        case 1:
            company = "SpaceX"
        case 2:
            company = "Virgin Galactic"
        }

        speed := rand.Intn(15) + 16                  // 16-30 km/s
        duration := distance / speed / secondsPerDay // days
        price := 20.0 + speed                        // millions

        if rand.Intn(2) == 1 {
            trip = "Round-trip"
            price = price * 2
        } else {
            trip = "One-way"
        }

        fmt.Printf("%-16v %4v %-10v $%4v\n", company, duration, trip, price)
    }
}
```

### 单元 2

#### 第 6 课

##### 实验: piggy.go

```
package main

import (
    "fmt"
    "math/rand"
)

func main() {
    piggyBank := 0.0

    for piggyBank < 20.00 {
        switch rand.Intn(3) {
        case 0:
            piggyBank += 0.05
        case 1:
            piggyBank += 0.10
        case 2:
            piggyBank += 0.25
        }
        fmt.Printf("$%5.2f\n", piggyBank)
    }
}
```

#### 第 7 课

##### 实验: piggy.go

```
package main

import (
    "fmt"
    "math/rand"
)

func main() {
    piggyBank := 0

    for piggyBank < 2000 {
        switch rand.Intn(3) {
        case 0:
            piggyBank += 5
        case 1:
            piggyBank += 10
        case 2:
            piggyBank += 25
        }

        dollars := piggyBank / 100
        cents := piggyBank % 100
        fmt.Printf("$%d.%02d\n", dollars, cents)
    }
}
```

#### 第 8 课

##### 实验: canis.go

```
package main

import (
    "fmt"
)

func main() {
    const distance = 236000000000000000
    const lightSpeed = 299792
    const secondsPerDay = 86400
    const daysPerYear = 365

    const years = distance / lightSpeed / secondsPerDay / daysPerYear

    fmt.Println("Canis Major Dwarf Galaxy is", years, "light years away.") *1*
}
```

+   ***1* 打印 Canis Major Dwarf Galaxy is 24962 light years away.**

#### 第 9 课

##### 实验: caesar.go

```
message := "L fdph, L vdz, L frqtxhuhg."

for i := 0; i < len(message); i++ {
    c := message[i]
    if c >= 'a' && c <= 'z' {
        c -= 3
        if c < 'a' {
            c += 26
        }
    } else if c >= 'A' && c <= 'Z' {
        c -= 3
        if c < 'A' {
            c += 26
        }
    }
    fmt.Printf("%c", c)
}
```

##### 实验: international.go

```
message := "Hola Estación Espacial Internacional"

for _, c := range message {
    if c >= 'a' && c <= 'z' {
        c = c + 13
        if c > 'z' {
            c = c - 26
        }
    } else if c >= 'A' && c <= 'Z' {
        c = c + 13
        if c > 'Z' {
            c = c - 26
        }
    }
    fmt.Printf("%c", c)
}
```

#### 第 10 课

##### 实验: input.go

```
yesNo := "1"

var launch bool

switch yesNo {
case "true", "yes", "1":
    launch = true
case "false", "no", "0":
    launch = false
default:
    fmt.Println(yesNo, "is not valid")
}

fmt.Println("Ready for launch:", launch)       *1*
```

+   ***1* 打印 Ready for launch: true**

#### 核心项目 11

##### 实验: decipher.go

```
cipherText := "CSOITEUIWUIZNSROCNKFD"
keyword := "GOLANG"
message := ""
keyIndex := 0

for i := 0; i < len(cipherText); i++ {
    // A=0, B=1, ... Z=25
    c := cipherText[i] - 'A'
    k := keyword[keyIndex] - 'A'

    // cipher letter - key letter
    c = (c-k+26)%26 + 'A'
    message += string(c)

    // increment keyIndex
    keyIndex++
    keyIndex %= len(keyword)
}

fmt.Println(message)
```

##### 实验: cipher.go

```
message := "your message goes here"
keyword := "golang"
keyIndex := 0
cipherText := ""

message = strings.ToUpper(strings.Replace(message, " ", "", -1))
keyword = strings.ToUpper(strings.Replace(keyword, " ", "", -1))

for i := 0; i < len(message); i++ {
    c := message[i]
    if c >= 'A' && c <= 'Z' {
        // A=0, B=1, ... Z=25
        c -= 'A'
        k := keyword[keyIndex] - 'A'

        // cipher letter + key letter
        c = (c+k)%26 + 'A'

        // increment keyIndex
        keyIndex++
        keyIndex %= len(keyword)
    }
    cipherText += string(c)
}
fmt.Println(cipherText)
```

### 单元 3

#### 第 12 课

##### 实验: functions.go

```
package main

import "fmt"

func kelvinToCelsius(k float64) float64 {
    return k - 273.15
}

func celsiusToFahrenheit(c float64) float64 {
    return (c * 9.0 / 5.0) + 32.0
}

func kelvinToFahrenheit(k float64) float64 {
    return celsiusToFahrenheit(kelvinToCelsius(k))
}

func main() {
    fmt.Printf("233° K is %.2f° C\n", kelvinToCelsius(233))
    fmt.Printf("0° K is %.2f° F\n", kelvinToFahrenheit(0))
}
```

#### 第 13 课

##### 实验: methods.go

```
package main

import "fmt"

type celsius float64

func (c celsius) fahrenheit() fahrenheit {
    return fahrenheit((c * 9.0 / 5.0) + 32.0)
}

func (c celsius) kelvin() kelvin {
    return kelvin(c + 273.15)
}

type fahrenheit float64

func (f fahrenheit) celsius() celsius {
    return celsius((f - 32.0) * 5.0 / 9.0)
}

func (f fahrenheit) kelvin() kelvin {
    return f.celsius().kelvin()
}

type kelvin float64

func (k kelvin) celsius() celsius {
    return celsius(k - 273.15)
}

func (k kelvin) fahrenheit() fahrenheit {
    return k.celsius().fahrenheit()
}

func main() {
    var k kelvin = 294.0
    c := k.celsius()
    fmt.Print(k, "° K is ", c, "° C")
}
```

#### 第 14 课

##### 实验: calibrate.go

```
package main

import (
    "fmt"
    "math/rand"
)

type kelvin float64
type sensor func() kelvin

func fakeSensor() kelvin {
    return kelvin(rand.Intn(151) + 150)
}

func calibrate(s sensor, offset kelvin) sensor {
    return func() kelvin {
        return s() + offset
    }
}

func main() {
    var offset kelvin = 5
    sensor := calibrate(fakeSensor, offset)

    for count := 0; count < 10; count++ {
        fmt.Println(sensor())
    }
}
```

#### 核心项目 15

##### 实验: tables.go

```
package main

import (
    "fmt"
)

type celsius float64

func (c celsius) fahrenheit() fahrenheit {
    return fahrenheit((c * 9.0 / 5.0) + 32.0)
}

type fahrenheit float64

func (f fahrenheit) celsius() celsius {
    return celsius((f - 32.0) * 5.0 / 9.0)
}

const (
    line         = "======================="
    rowFormat    = "| %8s | %8s |\n"
    numberFormat = "%.1f"
)

type getRowFn func(row int) (string, string)

// drawTable draws a two column table.
func drawTable(hdr1, hdr2 string, rows int, getRow getRowFn) {
    fmt.Println(line)
    fmt.Printf(rowFormat, hdr1, hdr2)
    fmt.Println(line)
    for row := 0; row < rows; row++ {
        cell1, cell2 := getRow(row)
        fmt.Printf(rowFormat, cell1, cell2)
    }
    fmt.Println(line)
}

func ctof(row int) (string, string) {
    c := celsius(row*5 - 40)
    f := c.fahrenheit()
    cell1 := fmt.Sprintf(numberFormat, c)
    cell2 := fmt.Sprintf(numberFormat, f)
    return cell1, cell2
}

func ftoc(row int) (string, string) {
    f := fahrenheit(row*5 - 40)
    c := f.celsius()
    cell1 := fmt.Sprintf(numberFormat, f)
    cell2 := fmt.Sprintf(numberFormat, c)
    return cell1, cell2
}

func main() {
    drawTable("°C", "°F", 29, ctof)
    fmt.Println()
    drawTable("°F", "°C", 29, ftoc)
}
```

### 单元 4

#### 第 16 课

##### 实验: chess.go

```
package main

import "fmt"

func display(board [8][8]rune) {
    for _, row := range board {
        for _, column := range row {
            if column == 0 {
                fmt.Print("  ")
            } else {
                fmt.Printf("%c ", column)
            }
        }
        fmt.Println()
    }
}

func main() {
    var board [8][8]rune

    // black pieces
    board[0][0] = 'r'
    board[0][1] = 'n'
    board[0][2] = 'b'
    board[0][3] = 'q'
    board[0][4] = 'k'
    board[0][5] = 'b'
    board[0][6] = 'n'
    board[0][7] = 'r'

    // pawns
    for column := range board[1] {
        board[1][column] = 'p'
        board[6][column] = 'P'
    }

    // white pieces
    board[7][0] = 'R'
    board[7][1] = 'N'
    board[7][2] = 'B'
    board[7][3] = 'Q'
    board[7][4] = 'K'
    board[7][5] = 'B'
    board[7][6] = 'N'
    board[7][7] = 'R'

    display(board)
}
```

#### 第 17 课

##### 实验: terraform.go

```
package main

import "fmt"

// Planets attaches methods to []string.
type Planets []string

func (planets Planets) terraform() {
    for i := range planets {
        planets[i] = "New " + planets[i]
    }
}

func main() {
    planets := []string{
        "Mercury", "Venus", "Earth", "Mars",
        "Jupiter", "Saturn", "Uranus", "Neptune",
    }
    Planets(planets[3:4]).terraform()
    Planets(planets[6:]).terraform()
    fmt.Println(planets)                      *1*
}
```

+   ***1* 打印 [Mercury Venus Earth New Mars Jupiter Saturn New Uranus New Neptune]**

#### 第 18 课

##### 实验: capacity.go

```
s := []string{}
lastCap := cap(s)

for i := 0; i < 10000; i++ {
    s = append(s, "An element")
    if cap(s) != lastCap {
        fmt.Println(cap(s))
        lastCap = cap(s)
    }
}
```

#### 第 19 课

##### 实验: words.go

```
package main

import (
    "fmt"
    "strings"
)

func countWords(text string) map[string]int {
    words := strings.Fields(strings.ToLower(text))
    frequency := make(map[string]int, len(words))
    for _, word := range words {
        word = strings.Trim(word, `.,"-`)
        frequency[word]++
    }
    return frequency
}

func main() {
    text := `As far as eye could reach he saw nothing but the stems of the
        great plants about him receding in the violet shade, and far overhead
        the multiple transparency of huge leaves filtering the sunshine to the
        solemn splendour of twilight in which he walked. Whenever he felt able
        he ran again; the ground continued soft and springy, covered with the
        same resilient weed which was the first thing his hands had touched in
        Malacandra. Once or twice a small red creature scuttled across his
        path, but otherwise there seemed to be no life stirring in the wood;
        nothing to fear -- except the fact of wandering unprovisioned and alone
        in a forest of unknown vegetation thousands or millions of miles
        beyond the reach or knowledge of man.`

    frequency := countWords(text)
    for word, count := range frequency {
        if count > 1 {
            fmt.Printf("%d %v\n", count, word)
        }
    }
}
```

#### 核心项目 20

##### 实验: life.go

```
package main

import (
    "fmt"
    "math/rand"
    "time"
)

const (
    width  = 80
    height = 15
)

// Universe is a two-dimensional field of cells.
type Universe [][]bool

// NewUniverse returns an empty universe.
func NewUniverse() Universe {
    u := make(Universe, height)
    for i := range u {
        u[i] = make([]bool, width)
    }
    return u
}

// Seed random live cells into the universe.
func (u Universe) Seed() {
    for i := 0; i < (width * height / 4); i++ {
        u.Set(rand.Intn(width), rand.Intn(height), true)
    }
}

// Set the state of the specified cell.
func (u Universe) Set(x, y int, b bool) {
    u[y][x] = b
}

// Alive reports whether the specified cell is alive.
// If the coordinates are outside of the universe, they wrap around.
func (u Universe) Alive(x, y int) bool {
    x = (x + width) % width
    y = (y + height) % height
    return u[y][x]
}

// Neighbors counts the adjacent cells that are alive.
func (u Universe) Neighbors(x, y int) int {
    n := 0
    for v := -1; v <= 1; v++ {
        for h := -1; h <= 1; h++ {
            if !(v == 0 && h == 0) && u.Alive(x+h, y+v) {
                n++
            }
        }
    }
    return n
}

// Next returns the state of the specified cell at the next step.
func (u Universe) Next(x, y int) bool {
    n := u.Neighbors(x, y)
    return n == 3 || n == 2 && u.Alive(x, y)
}

// String returns the universe as a string.
func (u Universe) String() string {
    var b byte
    buf := make([]byte, 0, (width+1)*height)

    for y := 0; y < height; y++ {
        for x := 0; x < width; x++ {
            b = ' '
            if u[y][x] {
                b = '*'
            }
            buf = append(buf, b)
        }
        buf = append(buf, '\n')
    }

    return string(buf)
}

// Show clears the screen and displays the universe.
func (u Universe) Show() {
    fmt.Print("\x0c", u.String())
}

// Step updates the state of the next universe (b) from
// the current universe (a).
func Step(a, b Universe) {
    for y := 0; y < height; y++ {
        for x := 0; x < width; x++ {
            b.Set(x, y, a.Next(x, y))
        }
    }
}

func main() {
    a, b := NewUniverse(), NewUniverse()
    a.Seed()

    for i := 0; i < 300; i++ {
        Step(a, b)
        a.Show()
        time.Sleep(time.Second / 30)
        a, b = b, a // Swap universes
    }
}
```

### 单元 5

#### 第 21 课

##### 实验: landing.go

```
type location struct {
    Name string  `json:"name"`
    Lat  float64 `json:"latitude"`
    Long float64 `json:"longitude"`

}
locations := []location{
    {Name: "Bradbury Landing", Lat: -4.5895, Long: 137.4417},
    {Name: "Columbia Memorial Station", Lat: -14.5684, Long: 175.472636},
    {Name: "Challenger Memorial Station", Lat: -1.9462, Long: 354.4734},
}

bytes, err := json.MarshalIndent(locations, "", "  ")
if err != nil {
    fmt.Println(err)
    os.Exit(1)
}

fmt.Println(string(bytes))
```

#### 第 22 课

##### 实验: landing.go

```
package main

import "fmt"

// location with a latitude, longitude.
type location struct {
    lat, long float64
}

// coordinate in degrees, minutes, seconds in a N/S/E/W hemisphere.
type coordinate struct {
    d, m, s float64
    h       rune
}

// newLocation from latitude, longitude d/m/s coordinates.
func newLocation(lat, long coordinate) location {
    return location{lat.decimal(), long.decimal()}
}

// decimal converts a d/m/s coordinate to decimal degrees.
func (c coordinate) decimal() float64 {
    sign := 1.0
    switch c.h {
    case 'S', 'W', 's', 'w':
        sign = -1
    }
    return sign * (c.d + c.m/60 + c.s/3600)
}

func main() {
    spirit := newLocation(coordinate{14, 34, 6.2, 'S'}, coordinate{175, 28,
21.5, 'E'})
    opportunity := newLocation(coordinate{1, 56, 46.3, 'S'}, coordinate{354,
28, 24.2, 'E'})
    curiosity := newLocation(coordinate{4, 35, 22.2, 'S'}, coordinate{137,
26, 30.12, 'E'})
    insight := newLocation(coordinate{4, 30, 0.0, 'N'}, coordinate{135, 54,
0, 'E'})
    fmt.Println("Spirit", spirit)
    fmt.Println("Opportunity", opportunity)
    fmt.Println("Curiosity", curiosity)
    fmt.Println("InSight", insight)
}
```

##### 实验: distance.go

```
package main

import (
    "fmt"
    "math"
)

// location with a latitude, longitude.
type location struct {
    lat, long float64
}

// coordinate in degrees, minutes, seconds in a N/S/E/W hemisphere.
type coordinate struct {
    d, m, s float64
    h       rune
}

// newLocation from latitude, longitude d/m/s coordinates.
func newLocation(lat, long coordinate) location {
    return location{lat.decimal(), long.decimal()}
}

// decimal converts a d/m/s coordinate to decimal degrees.
func (c coordinate) decimal() float64 {
    sign := 1.0
    switch c.h {
    case 'S', 'W', 's', 'w':
        sign = -1
    }
    return sign * (c.d + c.m/60 + c.s/3600)
}

// world with a volumetric mean radius in kilometers
type world struct {
    radius float64
}

// distance calculation using the Spherical Law of Cosines.
func (w world) distance(p1, p2 location) float64 {
    s1, c1 := math.Sincos(rad(p1.lat))
    s2, c2 := math.Sincos(rad(p2.lat))
    clong := math.Cos(rad(p1.long - p2.long))
    return w.radius * math.Acos(s1*s2+c1*c2*clong)
}

// rad converts degrees to radians.
func rad(deg float64) float64 {
    return deg * math.Pi / 180
}

var (
    mars  = world{radius: 3389.5}
    earth = world{radius: 6371}
)

func main() {
    spirit := newLocation(coordinate{14, 34, 6.2, 'S'}, coordinate{175, 28,
21.5, 'E'})
    opportunity := newLocation(coordinate{1, 56, 46.3, 'S'}, coordinate{354,
28, 24.2, 'E'})
    curiosity := newLocation(coordinate{4, 35, 22.2, 'S'}, coordinate{137,
26, 30.12, 'E'})
    insight := newLocation(coordinate{4, 30, 0.0, 'N'}, coordinate{135, 54,
0.0, 'E'})
    fmt.Printf("Spirit to Opportunity %.2f km\n", mars.distance(spirit,
opportunity))
    fmt.Printf("Spirit to Curiosity %.2f km\n", mars.distance(spirit,
curiosity))
    fmt.Printf("Spirit to InSight %.2f km\n", mars.distance(spirit, insight))

    fmt.Printf("Opportunity to Curiosity %.2f km\n", mars.distance(opportunity,
curiosity))
    fmt.Printf("Opportunity to InSight %.2f km\n", mars.distance(opportunity,
insight))

    fmt.Printf("Curiosity to InSight %.2f km\n", mars.distance(curiosity,
insight))

    london := newLocation(coordinate{51, 30, 0, 'N'}, coordinate{0, 8, 0, 'W'})
    paris := newLocation(coordinate{48, 51, 0, 'N'}, coordinate{2, 21, 0, 'E'})
    fmt.Printf("London to Paris %.2f km\n", earth.distance(london, paris))

    edmonton := newLocation(coordinate{53, 32, 0, 'N'}, coordinate{113, 30, 0,
'W'})
    ottawa := newLocation(coordinate{45, 25, 0, 'N'}, coordinate{75, 41, 0,
'W'})
    fmt.Printf("Hometown to Capital %.2f km\n", earth.distance(edmonton,
ottawa))

    mountSharp := newLocation(coordinate{5, 4, 48, 'S'}, coordinate{137, 51, 0,
'E'})
    olympusMons := newLocation(coordinate{18, 39, 0, 'N'}, coordinate{226, 12,
0, 'E'})
    fmt.Printf("Mount Sharp to Olympus Mons %.2f km\n",
mars.distance(mountSharp, olympusMons))
}
```

#### 第 23 课

##### 实验: gps.go

```
package main

import (
    "fmt"
    "math"
)

type world struct {
    radius float64
}

type location struct {
    name      string
    lat, long float64
}

func (l location) description() string {
    return fmt.Sprintf("%v (%.1f°, %.1f°)", l.name, l.lat, l.long)
}

type gps struct {
    world       world
    current     location
    destination location
}

func (g gps) distance() float64 {
    return g.world.distance(g.current, g.destination)
}

func (g gps) message() string {
    return fmt.Sprintf("%.1f km to %v", g.distance(),
g.destination.description())
}

func (w world) distance(p1, p2 location) float64 {
    s1, c1 := math.Sincos(rad(p1.lat))
    s2, c2 := math.Sincos(rad(p2.lat))
    clong := math.Cos(rad(p1.long - p2.long))
    return w.radius * math.Acos(s1*s2+c1*c2*clong)
 }

func rad(deg float64) float64 {
    return deg * math.Pi / 180
}

type rover struct {
    gps
}

func main() {
    mars := world{radius: 3389.5}
    bradbury := location{"Bradbury Landing", -4.5895, 137.4417}
    elysium := location{"Elysium Planitia", 4.5, 135.9}

    gps := gps{
        world:       mars,
        current:     bradbury,
        destination: elysium,
    }

    curiosity := rover{
        gps: gps,
    }

    fmt.Println(curiosity.message())           *1*
 }
```

+   ***1* 打印 545.4 km to Elysium Planitia (4.5°, 135.9°)**

#### 第 24 课

##### 实验: marshal.go

```
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

// coordinate in degrees, minutes, seconds in a N/S/E/W hemisphere.
type coordinate struct {
    d, m, s float64
    h       rune
}

// String formats a DMS coordinate.
func (c coordinate) String() string {
    return fmt.Sprintf("%v°%v'%.1f\" %c", c.d, c.m, c.s, c.h)
}

// decimal converts a d/m/s coordinate to decimal degrees.
func (c coordinate) decimal() float64 {
    sign := 1.0
    switch c.h {
    case 'S', 'W', 's', 'w':
        sign = -1
    }
    return sign * (c.d + c.m/60 + c.s/3600)
}

func (c coordinate) MarshalJSON() ([]byte, error) {
    return json.Marshal(struct {
        DD  float64 `json:"decimal"`
        DMS string  `json:"dms"`
        D   float64 `json:"degrees"`
        M   float64 `json:"minutes"`
        S   float64 `json:"seconds"`
        H   string  `json:"hemisphere"`
    }{
        DD:  c.decimal(),
        DMS: c.String(),
        D:   c.d,
        M:   c.m,
        S:   c.s,
        H:   string(c.h),
    })
}

// location with a latitude, longitude in decimal degrees.
type location struct {
    Name string     `json:"name"`
    Lat  coordinate `json:"latitude"`
    Long coordinate `json:"longitude"`
}

func main() {
    elysium := location{
        Name: "Elysium Planitia",
        Lat:  coordinate{4, 30, 0.0, 'N'},
        Long: coordinate{135, 54, 0.0, 'E'},
    }

    bytes, err := json.MarshalIndent(elysium, "", "  ")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }

    fmt.Println(string(bytes))
}
```

#### 核心项目 25

##### 实验: animals.go

```
package main

import (
    "fmt"
    "math/rand"
    "time"
)

type honeyBee struct {
    name string
}

func (hb honeyBee) String() string {
    return hb.name
}

func (hb honeyBee) move() string {
    switch rand.Intn(2) {
    case 0:
        return "buzzes about"
    default:
        return "flies to infinity and beyond"
    }
}

func (hb honeyBee) eat() string {
    switch rand.Intn(2) {
    case 0:
        return "pollen"
    default:
        return "nectar"
    }
}

type gopher struct {
    name string
}

func (g gopher) String() string {
    return g.name
}

func (g gopher) move() string {
    switch rand.Intn(2) {
    case 0:
        return "scurries along the ground"
    default:
        return "burrows in the sand"
    }
}

func (g gopher) eat() string {
    switch rand.Intn(5) {
    case 0:
        return "carrot"
    case 1:
        return "lettuce"
    case 2:
        return "radish"
    case 3:
        return "corn"
    default:
        return "root"
    }
}

type animal interface {
    move() string
    eat() string
}

func step(a animal) {
    switch rand.Intn(2) {
    case 0:
        fmt.Printf("%v %v.\n", a, a.move())
    default:
        fmt.Printf("%v eats the %v.\n", a, a.eat())
    }
}

const sunrise, sunset = 8, 18

func main() {
    rand.Seed(time.Now().UnixNano())

    animals := []animal{
        honeyBee{name: "Bzzz Lightyear"},
        gopher{name: "Go gopher"},
    }

    var sol, hour int

    for {
        fmt.Printf("%2d:00 ", hour)
        if hour < sunrise || hour >= sunset {
            fmt.Println("The animals are sleeping.")
        } else {
            i := rand.Intn(len(animals))
            step(animals[i])
        }

        time.Sleep(500 * time.Millisecond)

        hour++
        if hour >= 24 {
            hour = 0
            sol++
            if sol >= 3 {
                break
            }
        }
    }
}
```

### 单元 6

#### 第 26 课

##### 实验: turtle.go

```
package main

import "fmt"

type turtle struct {
    x, y int
}

func (t *turtle) up() {
    t.y--
}

func (t *turtle) down() {
    t.y++
}

func (t *turtle) left() {
    t.x--
}

func (t *turtle) right() {
    t.x++
}

func main() {
    var t turtle
    t.up()
    t.up()
    t.left()
    t.left()
    fmt.Println(t)       *1*
    t.down()
    t.down()
    t.right()
    t.right()
    fmt.Println(t)       *2*
}
```

+   ***1* 打印 {-2 -2}**

+   ***2* 打印 {0 0}**

#### 第 27 课

##### 实验: knights.go

```
package main

import (
    "fmt"
)

type item struct {
    name string
}

type character struct {
    name     string
    leftHand *item
}

func (c *character) pickup(i *item) {
    if c == nil || i == nil {
        return
    }
    fmt.Printf("%v picks up a %v\n", c.name, i.name)
    c.leftHand = i
}

func (c *character) give(to *character) {
    if c == nil || to == nil {
        return
    }
    if c.leftHand == nil {
        fmt.Printf("%v has nothing to give\n", c.name)
        return
    }
    if to.leftHand != nil {
        fmt.Printf("%v's hands are full\n", to.name)
        return
    }
    to.leftHand = c.leftHand
    c.leftHand = nil
    fmt.Printf("%v gives %v a %v\n", c.name, to.name, to.leftHand.name)
}

func (c character) String() string {
    if c.leftHand == nil {
        return fmt.Sprintf("%v is carrying nothing", c.name)
    }
    return fmt.Sprintf("%v is carrying a %v", c.name, c.leftHand.name)
}

func main() {
    arthur := &character{name: "Arthur"}

    shrubbery := &item{name: "shrubbery"}
    arthur.pickup(shrubbery)                *1*

    knight := &character{name: "Knight"}
    arthur.give(knight)                     *2*

    fmt.Println(arthur)                     *3*
    fmt.Println(knight)                     *4*
}
```

+   ***1* 打印 Arthur picks up a shrubbery**

+   ***2* 打印 Arthur gives Knight a shrubbery**

+   ***3* 打印 Arthur is carrying nothing**

+   ***4* 打印 Knight is carrying a shrubbery**

#### 第 28 课

##### 实验: url.go

```
u, err := url.Parse("https://a b.com/")

if err != nil {
    fmt.Println(err)                          *1*
    fmt.Printf("%#v\n", err)                  *2*

    if e, ok := err.(*url.Error); ok {
        fmt.Println("Op:", e.Op)              *3*
        fmt.Println("URL:", e.URL)            *4*
        fmt.Println("Err:", e.Err)            *5*
    }
    os.Exit(1)
}

fmt.Println(u)
```

+   ***1* 打印 parse https://a b.com/: 主机名中存在无效字符 “ ”**

+   ***2* 打印 &url.Error{Op:“parse”, URL:“https://a b.com/”, Err:” “}**

+   ***3* 打印 Op: parse**

+   ***4* 打印 URL: https://a b.com/**

+   ***5* 打印 Err: invalid character “ ” in host name**

#### 核心项目 29

##### 实验: sudoku.go

```
package main

import (
    "errors"
    "fmt"
    "os"
)

const (
    rows, columns = 9, 9
    empty         = 0
)

// Cell is a square on the Sudoku grid.
type Cell struct {
    digit int8
    fixed bool
}

// Grid is a Sudoku grid.
type Grid [rows][columns]Cell

// Errors that could occur.
var (
    ErrBounds     = errors.New("out of bounds")
    ErrDigit      = errors.New("invalid digit")
    ErrInRow      = errors.New("digit already present in this row")
    ErrInColumn   = errors.New("digit already present in this column")
    ErrInRegion   = errors.New("digit already present in this region")
    ErrFixedDigit = errors.New("initial digits cannot be overwritten")
)

// NewSudoku makes a new Sudoku grid.
func NewSudoku(digits [rows][columns]int8) *Grid {
    var grid Grid
    for r := 0; r < rows; r++ {
        for c := 0; c < columns; c++ {
            d := digits[r][c]
            if d != empty {
                grid[r][c].digit = d
                grid[r][c].fixed = true
            }
        }
    }
    return &grid
}

// Set a digit on a Sudoku grid.
func (g *Grid) Set(row, column int, digit int8) error {
    switch {
    case !inBounds(row, column):
        return ErrBounds
    case !validDigit(digit):
        return ErrDigit
    case g.isFixed(row, column):
        return ErrFixedDigit
    case g.inRow(row, digit):
        return ErrInRow
    case g.inColumn(column, digit):
        return ErrInColumn
    case g.inRegion(row, column, digit):
        return ErrInRegion
    }

    g[row][column].digit = digit
    return nil
}

// Clear a cell from the Sudoku grid.
func (g *Grid) Clear(row, column int) error {
    switch {
    case !inBounds(row, column):
        return ErrBounds
    case g.isFixed(row, column):
        return ErrFixedDigit
    }

    g[row][column].digit = empty
    return nil

}

func inBounds(row, column int) bool {
    if row < 0 || row >= rows || column < 0 || column >= columns {
        return false
    }
    return true
}

func validDigit(digit int8) bool {
    return digit >= 1 && digit <= 9
}

func (g *Grid) inRow(row int, digit int8) bool {
    for c := 0; c < columns; c++ {
        if g[row][c].digit == digit {
            return true
        }
    }
    return false
}

func (g *Grid) inColumn(column int, digit int8) bool {
    for r := 0; r < rows; r++ {
        if g[r][column].digit == digit {
            return true
        }
    }
    return false
}

func (g *Grid) inRegion(row, column int, digit int8) bool {
    startRow, startColumn := row/3*3, column/3*3
    for r := startRow; r < startRow+3; r++ {
        for c := startColumn; c < startColumn+3; c++ {
            if g[r][c].digit == digit {
                return true
            }
        }
    }
    return false
}

func (g *Grid) isFixed(row, column int) bool {
    return g[row][column].fixed
}

func main() {
    s := NewSudoku([rows][columns]int8{
        {5, 3, 0, 0, 7, 0, 0, 0, 0},
        {6, 0, 0, 1, 9, 5, 0, 0, 0},
        {0, 9, 8, 0, 0, 0, 0, 6, 0},
        {8, 0, 0, 0, 6, 0, 0, 0, 3},
        {4, 0, 0, 8, 0, 3, 0, 0, 1},
        {7, 0, 0, 0, 2, 0, 0, 0, 6},
        {0, 6, 0, 0, 0, 0, 2, 8, 0},
        {0, 0, 0, 4, 1, 9, 0, 0, 5},
        {0, 0, 0, 0, 8, 0, 0, 7, 9},
    })

    err := s.Set(1, 1, 4)
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }

    for _, row := range s {
        fmt.Println(row)
    }
}
```

### 单元 7

#### 第 30 课

##### 实验: remove-identical.go

```
package main

import (
    "fmt"
)

func main() {
    c0 := make(chan string)
    c1 := make(chan string)
    go sourceGopher(c0)
    go removeDuplicates(c0, c1)
    printGopher(c1)
}

func sourceGopher(downstream chan string) {
    for _, v := range []string{"a", "b", "b", "c", "d", "d", "d", "e"} {
        downstream <- v
    }
    close(downstream)
}

func removeDuplicates(upstream, downstream chan string) {
    prev := ""
    for v := range upstream {
        if v != prev {
            downstream <- v
            prev = v
        }
    }
    close(downstream)
}

func printGopher(upstream chan string) {
    for v := range upstream {
        fmt.Println(v)
    }
}
```

##### 实验: split-words.go

```
package main

import (
    "fmt"
    "strings"
)

func main() {
    c0 := make(chan string)
    c1 := make(chan string)
    go sourceGopher(c0)
    go splitWords(c0, c1)
    printGopher(c1)
}

func sourceGopher(downstream chan string) {
    for _, v := range []string{"hello world", "a bad apple", "goodbye all"}
{

        downstream <- v
    }
    close(downstream)
}

func splitWords(upstream, downstream chan string) {
    for v := range upstream {
        for _, word := range strings.Fields(v) {
            downstream <- word
        }
    }
    close(downstream)
}

func printGopher(upstream chan string) {
    for v := range upstream {
        fmt.Println(v)
    }
}
```

#### 第 31 课

##### 实验: positionworker.go

```
package main

import (
    "fmt"
    "image"
    "time"
)

func main() {
    go worker()
    time.Sleep(5 * time.Second)
}

func worker() {
    pos := image.Point{X: 10, Y: 10}
    direction := image.Point{X: 1, Y: 0}
    delay := time.Second
    next := time.After(delay)
    for {
        select {
        case <-next:
            pos = pos.Add(direction)
            fmt.Println("current position is ", pos)
            delay += time.Second / 2
            next = time.After(delay)
        }
    }
}
```

##### 实验: rover.go

```
package main

import (
    "image"
    "log"
    "time"
)

func main() {
    r := NewRoverDriver()
    time.Sleep(3 * time.Second)
    r.Left()
    time.Sleep(3 * time.Second)
    r.Right()
    time.Sleep(3 * time.Second)
    r.Stop()
    time.Sleep(3 * time.Second)
    r.Start()
    time.Sleep(3 * time.Second)
}

// RoverDriver drives a rover around the surface of Mars.
type RoverDriver struct {
    commandc chan command
}

// NewRoverDriver starts a new RoverDriver and returns it.
func NewRoverDriver() *RoverDriver {
    r := &RoverDriver{
        commandc: make(chan command),
    }
    go r.drive()
    return r
}

type command int

const (
    right = command(0)
    left  = command(1)
    start = command(2)
    stop  = command(3)
)

// drive is responsible for driving the rover. It
// is expected to be started in a goroutine.
func (r *RoverDriver) drive() {
    pos := image.Point{X: 0, Y: 0}
    direction := image.Point{X: 1, Y: 0}
    updateInterval := 250 * time.Millisecond
    nextMove := time.After(updateInterval)
    speed := 1
    for {
        select {
        case c := <-r.commandc:
            switch c {
            case right:
                direction = image.Point{
                    X: -direction.Y,
                    Y: direction.X,
                }
            case left:
                direction = image.Point{
                    X: direction.Y,
                    Y: -direction.X,
                }
            case stop:
                speed = 0
            case start:
                speed = 1
            }
            log.Printf("new direction %v; speed %d", direction, speed)
        case <-nextMove:
            pos = pos.Add(direction.Mul(speed))
            log.Printf("moved to %v", pos)
            nextMove = time.After(updateInterval)
        }
    }
}

// Left turns the rover left (90° counterclockwise).
func (r *RoverDriver) Left() {
    r.commandc <- left
}

// Right turns the rover right (90° clockwise).
func (r *RoverDriver) Right() {
    r.commandc <- right
}

// Stop halts the rover.
func (r *RoverDriver) Stop() {
    r.commandc <- stop
}

// Start gets the rover moving.
func (r *RoverDriver) Start() {
    r.commandc <- start
}
```

#### 核心项目 32

##### 实验: lifeonmars.go

```
package main

import (
    "fmt"
    "image"
    "log"
    "math/rand"
    "sync"
    "time"
)

func main() {
    marsToEarth := make(chan []Message)
    go earthReceiver(marsToEarth)

    gridSize := image.Point{X: 20, Y: 10}
    grid := NewMarsGrid(gridSize)
    rover := make([]*RoverDriver, 5)
    for i := range rover {
        rover[i] = startDriver(fmt.Sprint("rover", i), grid, marsToEarth)
    }
    time.Sleep(60 * time.Second)
}

// Message holds a message as sent from Mars to Earth.
type Message struct {
    Pos       image.Point
    LifeSigns int
    Rover     string

}
const (
    // The length of a Mars day.
    dayLength = 24 * time.Second
    // The length of time per day during which
    // messages can be transmitted from a rover to Earth.
    receiveTimePerDay = 2 * time.Second
)

// earthReceiver receives messages sent from Mars.
// As connectivity is limited, it only receives messages
// for some time every Mars day.
func earthReceiver(msgc chan []Message) {
    for {
        time.Sleep(dayLength - receiveTimePerDay)
        receiveMarsMessages(msgc)
    }
}

// receiveMarsMessages receives messages sent from Mars
// for the given duration.
func receiveMarsMessages(msgc chan []Message) {
    finished := time.After(receiveTimePerDay)
    for {
        select {
        case <-finished:
            return
        case ms := <-msgc:
            for _, m := range ms {
                log.Printf("earth received report of life sign level %d from
%s at %v", m.LifeSigns, m.Rover, m.Pos)
            }
        }
    }
}

func startDriver(name string, grid *MarsGrid, marsToEarth chan []Message)
*RoverDriver {
    var o *Occupier
    // Try a random point; continue until we've found one that's
    // not currently occupied.
    for o == nil {
        startPoint := image.Point{X: rand.Intn(grid.Size().X), Y: rand.Intn(
grid.Size().Y)}
        o = grid.Occupy(startPoint)
    }
    return NewRoverDriver(name, o, marsToEarth)
}

// Radio represents a radio transmitter that can send
// message to Earth.
type Radio struct {
    fromRover chan Message
}

// SendToEarth sends a message to Earth. It always
// succeeds immediately - the actual message
// may be buffered and actually transmitted later.
func (r *Radio) SendToEarth(m Message) {
    r.fromRover <- m
}

// NewRadio returns a new Radio instance that sends
// messages on the toEarth channel.
func NewRadio(toEarth chan []Message) *Radio {
    r := &Radio{
        fromRover: make(chan Message),
    }
    go r.run(toEarth)
    return r
}

// run buffers messages sent by a rover until they
// can be sent to Earth.
func (r *Radio) run(toEarth chan []Message) {
    var buffered []Message
    for {
        toEarth1 := toEarth
        if len(buffered) == 0 {
            toEarth1 = nil
        }
        select {
        case m := <-r.fromRover:
            buffered = append(buffered, m)
        case toEarth1 <- buffered:
            buffered = nil
        }
    }
}

// RoverDriver drives a rover around the surface of Mars.
type RoverDriver struct {
    commandc chan command
    occupier *Occupier
    name     string
    radio    *Radio
}

// NewRoverDriver starts a new RoverDriver and returns it.
func NewRoverDriver(
    name string,
    occupier *Occupier,
    marsToEarth chan []Message,
) *RoverDriver {
    r := &RoverDriver{
        commandc: make(chan command),
        occupier: occupier,
        name:     name,
        radio:    NewRadio(marsToEarth),
    }
    go r.drive()
    return r
}

type command int

const (
    right command = 0
    left  command = 1
)

// drive is responsible for driving the rover. It
// is expected to be started in a goroutine.
func (r *RoverDriver) drive() {
    log.Printf("%s initial position %v", r.name, r.occupier.Pos())
    direction := image.Point{X: 1, Y: 0}
    updateInterval := 250 * time.Millisecond
    nextMove := time.After(updateInterval)
    for {
        select {
        case c := <-r.commandc:
            switch c {
            case right:
                direction = image.Point{
                    X: -direction.Y,
                    Y: direction.X,
                }
            case left:
                direction = image.Point{
                    X: direction.Y,
                    Y: -direction.X,
                }
            }
            log.Printf("%s new direction %v", r.name, direction)
        case <-nextMove:
            nextMove = time.After(updateInterval)
            newPos := r.occupier.Pos().Add(direction)
            if r.occupier.MoveTo(newPos) {
                log.Printf("%s moved to %v", r.name, newPos)
                r.checkForLife()
                break
            }
            log.Printf("%s blocked trying to move from %v to %v", r.name,
r.occupier.Pos(), newPos)
            // Pick one of the other directions randomly.
            // Next time round, we'll try to move in the new
            // direction.
            dir := rand.Intn(3) + 1
            for i := 0; i < dir; i++ {
                direction = image.Point{
                    X: -direction.Y,
                    Y: direction.X,
                }
            }
            log.Printf("%s new random direction %v", r.name, direction)
        }
    }
}

func (r *RoverDriver) checkForLife() {
    // Successfully moved to new position.
    sensorData := r.occupier.Sense()
    if sensorData.LifeSigns < 900 {
        return
    }
    r.radio.SendToEarth(Message{
        Pos:       r.occupier.Pos(),
        LifeSigns: sensorData.LifeSigns,
        Rover:     r.name,
    })
}

// Left turns the rover left (90° counterclockwise).
func (r *RoverDriver) Left() {
    r.commandc <- left
}

// Right turns the rover right (90° clockwise).
func (r *RoverDriver) Right() {
    r.commandc <- right
}

// MarsGrid represents a grid of some of the surface
// of Mars. It may be used concurrently by different
// goroutines.
type MarsGrid struct {
    bounds image.Rectangle
    mu     sync.Mutex
    cells  [][]cell
}

// SensorData holds information about what's in
// a point in the grid.
type SensorData struct {
    LifeSigns int
}

type cell struct {
    groundData SensorData
    occupier   *Occupier
}

// NewMarsGrid returns a new MarsGrid of the
// given size.
func NewMarsGrid(size image.Point) *MarsGrid {
    grid := &MarsGrid{
        bounds: image.Rectangle{
            Max: size,
        },
        cells: make([][]cell, size.Y),
    }
    for y := range grid.cells {
        grid.cells[y] = make([]cell, size.X)
        for x := range grid.cells[y] {
            cell := &grid.cells[y][x]
            cell.groundData.LifeSigns = rand.Intn(1000)
        }
    }
    return grid
}

// Size returns a Point representing the size of the grid.
func (g *MarsGrid) Size() image.Point {
    return g.bounds.Max
}

// Occupy occupies a cell at the given point in the grid. It
// returns nil if the point is already occupied or the point is outside
// the grid. Otherwise it returns a value that can be used
// to move to different places on the grid.
func (g *MarsGrid) Occupy(p image.Point) *Occupier {
    g.mu.Lock()
    defer g.mu.Unlock()
    cell := g.cell(p)
    if cell == nil || cell.occupier != nil {
        return nil
    }
    cell.occupier = &Occupier{
        grid: g,
        pos:  p,
    }
    return cell.occupier
}

func (g *MarsGrid) cell(p image.Point) *cell {
    if !p.In(g.bounds) {
        return nil
    }
    return &g.cells[p.Y][p.X]
}

// Occupier represents an occupied cell in the grid.
type Occupier struct {
    grid *MarsGrid
    pos  image.Point
}

// MoveTo moves the occupier to a different cell in the grid.
// It reports whether the move was successful
// It might fail because it was trying to move outside
// the grid or because the cell it's trying to move into
// is occupied. If it fails, the occupier remains in the same place.
func (o *Occupier) MoveTo(p image.Point) bool {
    o.grid.mu.Lock()
    defer o.grid.mu.Unlock()
    newCell := o.grid.cell(p)
    if newCell == nil || newCell.occupier != nil {
        return false
    }
    o.grid.cell(o.pos).occupier = nil
    newCell.occupier = o
    o.pos = p
    return true
}

// Sense returns sensory data from the current cell.
func (o *Occupier) Sense() SensorData {
    o.grid.mu.Lock()
    defer o.grid.mu.Unlock()
    return o.grid.cell(o.pos).groundData
}

// Pos returns the current grid position of the occupier.
func (o *Occupier) Pos() image.Point {
    return o.pos
}
```
