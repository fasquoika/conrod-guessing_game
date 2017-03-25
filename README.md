# Attention!
There were breaking changes in conrod from version 0.49.0 to 0.51.1. 
I'll update the turorial sooner or later, till then it works with 
v0.49.0.

## Using Conrod - Let's build something simple!

We're about to make a simple application step by step. The example app will be a graphical version of the Guessing Game. I'm 
pretty sure you came across this one in the [Rust Book](https://doc.rust-lang.org/book/guessing-game.html). This time, we're 
going to make it look 'pretty'!
![Guessing Game UI](/illustration/app_ui.png)


But before we move on, here's an overview on how I'm going to structure the application based on how `conrod` and IMGUIs work.
### How to think IMGUI:
In an IMGUI library widgets do not hold state. They are recreated with the data provided at each update cycle. This means you 
don't have to manually update the content of the widgets. Take this counter example in Visual C#:
```cs
private void ButtonInc_Click(object sender, EventArgs e)
{
    count += 1;
    LabelCount.Text = count.ToString();
}
```
First, the update is tied to the `callback` of the `ButtonInc` object and we tell the `LabelCount` object to update its `Text`
field. At each point in the code that you change `count`, you also have to update the label. For convenience, you can 
encapsulate these changes into a single function to call at each point `count` changes, like so:
```cs
private void UpdateCount(int amount)
{
    count += amount;
    LabelCount.Text = count.ToString();
}
```
If you think about it, this is a little inefficient. You store both `count` and `LabelCount.Text` even though they represent 
the same thing. In `conrod` the code would look like this:
```rust
let mut count = 0;
//... stuff here
for click in Button::new()
    .options_come_here() //all widget settings will be defined here, like size, color...
{
    count += 1;
}

Text::new(&count.to_string()).options_come_here();
```
An IMGUI is a bit more descriptive, what this snippet says is "at each click increment the counter, and show it to me as 
text". Let's say you have a dozen buttons. Each adds, subtracts or multiplies `count`. You still do the `count.to_string()` 
once and only once, when you redraw the `Text`. This is in stark contrast to always calling `UpdateCount()`, which is what you 
would have to do in the  C# example.

The idea is that the **content** you see on screen is driven by the **logic** of the program, while the logic is driven by the 
application **data**! The separation in the application that I'm going to use will be the same. There's going to be a 
`main.rs` with all the setups and inits for showing the **content**. There's also going to be a `logic.rs` for the program 
logic, and an `app.rs` with the application data and some helper functions which don't necessarily belong to the other 2 
categories.

##### So, remember:

![IMGUI content flow](/illustration/imgui_content_flow.png)

**Note 1:** in C# there's also a way to use the same function for multpile callbacks and then checking who's the sender and 
performing the appropiate computations, so you can call `UpdateCount()` only once.

**Note 2:** IMGUIs are not perfect for everything, but they're great for displaying live data and having a reactive UI, ie. in 
game UI or for displaying sensor data, soundwave forms.

### 0. Setting up the project:
Type `cargo new --bin guessing_game` into your terminal to get the skeleton of the project. First we need to add the 
dependencies for our application, so open up the `Cargo.toml` file in the freshly created directory and lets add a few lines 
to it!

```toml
[dependencies]
conrod = "0.49.0"
find_folder="*"
rand="*"
```

We need to fetch `conrod` from [crates.io](https://crates.io/crates/conrod), and we're going to do that by adding it to the 
dependencies. `"*"` marks we want to get the latest version. `find_folder` can be used to load resources, like pictures and 
sounds, but the most basic example is fonts. `rand` will be used to generate the 'secret' number, with a constant number it 
would be a silly game, don't you think?

As it can take a while to download and compile for the first time, let's do the heavy lifting before we start coding, so type 
`cargo run` into your terminal to fetch and compile all necessary packages. If everything is fine, you'll see `Hello, world!` 
printed out on the last line.

All cool? Great! Let's move on!

### 1. Basic window
In this section we will write all the code needed to get an empty window to show up. For now we'll use `piston_window` as the 
backend.

##### Imports:
Open up your `main.rs` file and edit to make it look like this:
```rust
#[macro_use]
extern crate conrod;
extern crate find_folder;
extern crate rand;

fn main () {
    use conrod::backend::piston::{self,Window,WindowEvents,OpenGL};
    use conrod::backend::piston::event::UpdateEvent;

    /*
    ...
    */
}
```
For now, that's all we need to import. The rest is going to be inside functions and modules.
You see that `#[macro_use]` up top, right? `conrod` has a macro wich is always used, `widget_ids!`. Wait until we start 
working with widgets, I'll tell you more.

##### Definitions:
The 3 basic things we will need are the title, the width, and the height of the window.
```rust
fn main() {
/* import here */

	let width: u32 = 450;
    let height: u32 = 350;
    let title = "Guessing Game";

}
```

To get a nice window we are going to use something called 'method chaining' or 'the builder pattern'. Think about it like 
ordering some food! You first tell the cashier you want pizza, than you say "Yo, I love spicey food, add some pepperoni, and 
coke too".
```rust
let my_dinner = Restaurant::Order::new()
	.food("pizza")
	.with_extra("pepperoni")
	.drink("coke")
	.done();
```

This way we're going to tell `conrod` with what parameters the window should be created.

```rust
 let mut window: Window = piston::window::WindowSettings::new(title, [width, height])
        .opengl(OpenGL::V3_2)
        .exit_on_esc(true)
        .vsync(true)
        .build()
        .unwrap();
```
Let's go through what we've specified here!
* `WindowSettings::new()` returns with a 'builder' for the `Window` type containing a bunch of defaults, which we can modify 
with the rest of the functions!
* `.opengl()` sets the version of OpenGL the backend should use for the rendering. You can choose from a predefined set of 
versions. To check out from what you can choose from, click [here](http://docs.piston.rs/conrod/conrod/backend/piston/enum.OpenGL.html).
* `.exit_on_esc()` makes it possible for you to close the window by pressing, well, the `esc` key.
* `.vsync()` can help you get rid of screen tearing during resize events.
* `.build()` is the function that actually does the job and returns with either a `Window` or some error message.
* `.unwrap()`, as you probably know, gets the value out of a `Result`, and panics if there is an error

You can take a look at all the other options that can be set [here](http://docs.piston.rs/conrod/conrod/backend/piston/window/struct.WindowSettings.html).

We also need to get the events from the window. For that we have to create an event loop iterator.
```rust
let mut events = WindowEvents::new();
```
##### The event loop:
The only thing left is to write a loop in which we can check the window events, and it also keeps are window open.
```rust
while let Some(event) = window.next_event(&mut events) {
    // stuff
}
```
This basically says 'while there is an event, I'll keep doing `{ //stuff }`', meaning this loop runs while the window is open.
We are going to use this loop to draw, update and do pretty much everything we need. But don't be confused! It doesn't mean 
the entire program will be an if-else spagetti. Remember the separation I talked at the beginning. :)

If you do `cargo run`, a black window should pop up with the given title and dimensions. Congrats! :)

### 2. Application and game data
In this section our goal is to define the kind of data we are going to track. This is something you may not have to do in 
traditional, retained mode GUI systems as widgets store much of that data. But remember `conrod` is an IMGUI library of which 
the most notable feature is that widgets do not store state! For instance `TextBox` doesn't store the string it displays, so 
it basically writes into a variable, and reads back from a variable at each update. The window data is stored, so you don't 
have to pass `window`, or its size and title to the draw function. Writing a function to accept a struct is still easier than 
adding more and more parameters and then fighting with the borrow checker, I think.

Taking care of widget state is sometimes a problem, but most of the time it is a great way of specifing no more than what we 
need!

So let's create an `app.rs` file and move all the **data** from `main` if it's not necessary for the **content** to be displayed. In `app` we have:
```rust
pub struct AppData {
    pub width: u32,
    pub height: u32,
    pub guess: String,
    pub title: String,
    pub info: String,
    // fields are public, so less `fn` to implement
    // for control over data manipulation feel free to extend the `impl` block
}

impl AppData {
    pub fn new(width: u32, height: u32, title: &str) -> AppData {
        AppData {
            width: width,
            height: height,
            guess: String::new(),
            title: title.to_owned(),
            info: "? X".to_owned(),
        }
    }

    pub fn new_guess(&mut self, guess: &str) {
        self.guess = guess.to_owned();
    }
}
```
If you remember the design up top, it had a `Button`, a `TextBox` for entering the guess and a `Text` for information about the previous guess.
`info` is used for the `Text` and `guess` is used for the `TextBox`'s content. There was also one more `Text` for showing the guesses left, the data for that is going to be in an other struct, as it has more to do with the rules of the game.

Now we define all the data for the game, and the behaviour aka. rules of the game.
```rust
pub struct GameData {
    secret_num: i32,
    no_guess: i32,
    pub range_min: i32,
    pub range_max: i32,
    win: bool,
}

impl GameData {
    pub fn new(guess_num: i32, range: [i32; 2]) -> Self {
        use super::rand::{self, Rng};
	// in the fn signature I use [i32;2] because for me [num0,num1] resembels a closed interval, like in math
    // change to (guess_num, min, max) format if you'd like to
        let (min, max) = (range[0], range[1]);
        GameData {
            secret_num: rand::thread_rng().gen_range(min, max + 1),
            no_guess: guess_num,
            range_min: min,
            range_max: max,
            win: false,
        }
    }
	// to make it impossible to overwrite it outside of the rules
    pub fn get_no_guess(&self) -> i32 {
        self.no_guess
    }
	// add get() and set() for range bounds if you want to, I don't really...

	// defines how you can modify the no_guess and the rest of the internals
    pub fn new_guess(&mut self, guess: &str) -> String {
        if guess != "" {
            let g: i32 = guess.parse().expect("`guess` cannot be converted to number");
            if g == self.secret_num {
                self.win = true;
                return "= X".to_owned();
            } else {
                self.no_guess -= 1;
                return if g < self.secret_num {
                    "< X".to_owned()
                } else {
                    "> X".to_owned()
                };
            }
        }
        "? X".to_owned()
    }
    // easiest way to concat converted values and do the formatting
    pub fn show_range(&self) -> String {
        format!("({}, {})", self.range_min, self.range_max)
    }

    pub fn end(&self) -> bool {
        self.win || self.no_guess == 0
    }

    pub fn restart(&mut self) {
        use super::rand::{self, Rng};

        self.secret_num = rand::thread_rng().gen_range(self.range_min, self.range_max + 1);
        self.win = false;
        self.no_guess = 10;
    }
}
```

The modifications in `main.rs` to make:
```rust
/* externs */

mod app;

fn main() {
    /* imports */

    // mut so later we can set a few things
    let mut data = app::AppData::new(450, 350, "Guessing Game");
    let mut game = app::GameData::new(10, [1, 50]);

    let mut window: Window = piston::window::WindowSettings::new(data.title.clone(), [data.width, data.height])
        .opengl(OpenGL::V3_2)
        .exit_on_esc(true)
        .vsync(true)
        .build()
        .unwrap();

    /* event loop iterator, event loop */
}
```

### 3. UI setups
The next step is going to be creating the `Ui`. This is special as `Ui` in `conrod` is the data structure that keeps track of 
the state of every `Widget`, the `Theme` you use globally (this is for in app themeing, does not support OS theme at the 
moment) and much more. In general, `Ui` handles draw events, like highlighting a hovered button. It also tries to reduce the 
number of draw calls, meaning it updates the screen if and only if there's an update which demands a redraw. (Of course, you 
can demand continuous redraws!)

For more, go look at [this](http://docs.piston.rs/conrod/conrod/struct.Ui.html)!

##### UiBuilder:
`UiBuilder` lets you specify the ui's dimensions, theme (if none then the default is used) and widget capacity.
As the `Ui` tracks widgets in a graph, it can grow organically or you can help it by predefineing how many widgets you're 
about to instantiate. This could help app launch times, so you're not expanding the graph dynamically in the very first draw 
cycle.

Right now, lets stick with the dimensions only, so you can experiment with adding more to what I've done.

```rust
// add this to main()
let mut ui = conrod::UiBuilder::new([WIDTH as f64,HEIGHT as f64]).build();
```
##### Ids:
As the widget states are stored inside the graph, we use `Ids` to acces specific widgets. `conrod` provides a macro for 
creating an `Ids` struct with the given fields. In my opinion `Ids` are part of the **data**, so place the macro in `app.rs`.
```rust
widget_ids! {
    pub struct Ids {
        canvas,
        button,
        count_text,
        info_text,
        textbox,
    }
}
/* AppData and GameData */
```
So, the macro will create the `Ids` struct for us, with fields with the name representing a unique widget id. These `Ids` need 
a way of knowing if they exist in the `ui`. If you add more widgets, don't forget to extend `Ids`.
```rust
// add this to main()
let mut ids = Ids::new(ui.widget_id_generator());
```

I think the names are descriptive enough. The only thing to mention is that `ids.canvas` is the id of `Canvas`, a widget that 
can hold other widgets and usually fills the entire window.

### 4. Content rendering
Next step is to finally add something to our draw loop!
```rust
while let Some(event) = window.next_event(&mut events) {
        event.update(|_| logic::update(ui.set_widgets(), &ids, &mut game, &mut data));
}
```
Let me skip ahead without saying anything, and let's create the `logic::update()` just to see something on screen. This 
functions is going to be the **logic**. You know what to do! Create `logic.rs` and place this snippet:
```rust
use app::{GameData, AppData, Ids};
use super::conrod;

pub fn update(ref mut ui: conrod::UiCell, ids: &Ids, game: &mut GameData, data: &mut AppData) {
    use conrod::{Colorable, Labelable, Positionable, Sizeable};
    use conrod::Widget;
    use conrod::widget::text_box;
    use conrod::widget::{Canvas, Button, Text, TextBox};

    let mut caption = "Guess number between ".to_owned();
    let range = game.show_range();
    caption.push_str(&range);

    Canvas::new()
        .color(conrod::color::WHITE)
        .title_bar(&caption)
        .pad(40.0)
        .set(ids.canvas, ui);
}
```

If you run it, you see that it's still a black window, well... Every cycle you tell `conrod` to update the `canvas` in the 
state graph, but have you ever rendered the graph (a.k.a. the `ui`)? Nope, so let's do it!

```rust
let mut text_texture_cache = piston::window::GlyphCache::new(&mut window,0,0);
let image_map = conrod::image::Map::new();

while let Some(event) = window.next_event(&mut events) {
	// update ui first
	event.update(|_| set_ui(ui.set_widgets(),&ids));
	// then display
	window.draw_2d(&event, |c,g| {
		if let Some(primitives) = ui.draw_if_changed() {
			fn texture_from_image<T>(img:&T) -> &T { img };
			piston::window::draw(c,g, primitives,
								 &mut text_texture_cache,
								 &image_map,
								 texture_from_image);
		}
	});
}
```
`text_texture_cache` is used to efficiently cache fonts. `image_map` describes the widget-image relationships. Confused? No 
worries, right now only `text_texture_cache` will be used, the other one is just an empty shell to feed into the `draw()` 
function of `piston` backend.

If you run the application now, you'll see a window popping up filled with white. So the canvas actually fills out the window, 
huh great! Let's test resizing the window. Oooops, you quickly realize that it's not working properly. Also, where's the 
`caption`?

##### Resize:
The canvas has the initial width and height of the window, and it doesn't grow beyound. It's because `conrod` supports 
multiple backends, so you have to tell it what kind of event conversions should be done. Add 3 more lines right above 
`event.update()` to make resizing work and generally register window events.
```rust
if let Some(e) = piston::window::convert_event(event.clone(), &window) {
            ui.handle_event(e);
}
```
##### Fonts:
So text isn't rendered as there's no font defined in `ui` by default. I would say fonts are also part of **data**, so write 
this helper function to load fonts in `app.rs`:
```rust
pub fn load_font(font: &str) -> super::std::path::PathBuf {
    use super::find_folder::Search::KidsThenParents;

    let fonts_dir = KidsThenParents(3, 5).for_folder("fonts").expect("`fonts/` not found!");
    let font_path = fonts_dir.join(font);

    font_path
}
```
It assumes that you have a the font located in the project directory in the folder `*/fonts/`.

Let's load the font!
```rust
/* data and game inits above, just to organize data in one place */
let font_path = app::load_font("UbuntuMono-R.ttf");

//stuff

let mut ui = conrod::UiBuilder::new([data.width as f64, data.height as f64]).build();
ui.fonts.insert_from_file(font_path).unwrap();
let ids = Ids::new(ui.widget_id_generator());
```

If you run the application now, you'll get the title and resize will work properly.

### 5. Logic

I already asked you to skip ahead and write this code in `logic.rs`, the only thing left is to walk you through!

```rust
use app::{GameData, AppData, Ids};
use super::conrod;

pub fn update(ref mut ui: conrod::UiCell, ids: &Ids, game: &mut GameData, data: &mut AppData) {
    use conrod::{Colorable,Labelable, Positionable, Sizeable};
    use conrod::Widget;
    use conrod::widget::text_box;
    use conrod::widget::{Canvas, Button, Text, TextBox};

    let caption = app::set_caption(&game);

    Canvas::new()
        .color(conrod::color::WHITE)
        .title_bar(&caption)
        .pad(40.0)
        .set(ids.canvas, ui);
}
```
Where `set_caption()` is in `app.rs` and defined as:
```rust
pub fn set_caption(game: &GameData) -> String {
    let mut caption = "Guess number between ".to_owned();
    let range = game.show_range();
    caption.push_str(&range);

    caption
}
```
Q: **Why do I define the `caption` here?**
A: Because it's the combination of the two states, it does not belongs to any of them, only to the canvas.

So after all this code, I'm going to start with `use ...`. Let's breakdown each:
* [Colorable](http://docs.piston.rs/conrod/conrod/color/trait.Colorable.html) - lets you use colors on widgets, and defines 
default colors, like `WHITE` in the canvas definition. Also has ways of creating colors from RGB and HSL.
* [Labelable](http://docs.piston.rs/conrod/conrod/trait.Labelable.html) - lets you use functions to place labels on widgets 
like `Button`
* [Positionable](http://docs.piston.rs/conrod/conrod/trait.Positionable.html) - lets you use functions to move widgets around, 
align them to others, etc.
* [Sizeable](http://docs.piston.rs/conrod/conrod/trait.Sizeable.html) - lets you get and set the size of widgets at a given 
update cycle
* the rest are just imports for default and specific widget behaviours

What you see in the type signature is that we interact with the `ui` through something called the `UiCell`. The `UiCell` 
restricts you from mutating the `ui` in ways you shouldn't do, and provides methods on how you should. You can dig through all 
of what `UiCell` can do [here](http://docs.piston.rs/conrod/conrod/struct.UiCell.html).

Let's look at the `Canvas` for a second! As you can see we're not binding a canvas object to a variable, we simply declare 
what kind of canvas we want. I say, let's make it white, I want the padding from all edges to be 40, and this is going to live 
under the name `canvas` with a title defined by `caption`. You can do [many more](http://docs.piston.rs/conrod/conrod/widget/canvas/struct.Canvas.html).

The most important bit is `set()`. It creates the widget, adds it to the `ui` graph with the given id.

You can look up `Positionable` to see all the functions available, and to have a better understanding of what the code below 
means. I'll give you the basic syntax for each widget used, and then the final logic and one more 'homework' problem to do.
##### Button:
```rust
for _click in Button::new()
    .top_left_of(ids.canvas)
    .w_h(100.0, 50.0)
    .label("Guess!")
	.set(ids.button, ui)
{
	data.info = game.new_guess(&data.guess);
}
```
It reads quite well: 'I have a button at the top left corner of the canvas, with size of 100 by 50, it has the label "Guess". 
For each click this button takes a new guess and based on that updates the info.' Before you start screaming, that `button` 
isn't at the top left, remember we have an edge padding of 40 set for the `canvas`. `_` in `_click` is used to tell the 
compiler we named the iterator, but not going to use it, and we will get no warnings.

##### Text:
```rust
Text::new(&(game.get_no_guess().to_string()))
    .middle_of(ids.button)
    .down_from(ids.button, 10.0)
    .set(ids.count_text, ui);
```

##### TextBox:
```rust
for edit in TextBox::new(&data.guess)
    .align_text_middle()
    .right_from(ids.button, 10.0)
    .w_h(200.0, 50.0)
    .set(ids.textbox, ui)
{
    match edit {
        text_box::Event::Enter => {
            data.info = game.new_guess(&data.guess);
        }
        text_box::Event::Update(text) => {
            data.new_guess(&text);
        }
    }
}
```

The complete update logic is:
```rust
pub fn update(/* ... */) {
    /* imports, caption and canvas */

	if !game.end() {
        for _click in Button::new()
            .top_left_of(ids.canvas)
            .w_h(100.0, 50.0)
            .label("Guess!")
            .set(ids.button, ui) {
            data.info = game.new_guess(&data.guess);
        }

        Text::new(&(game.get_no_guess().to_string()))
            .middle_of(ids.button)
            .down_from(ids.button, 10.0)
            .set(ids.count_text, ui);

        for edit in TextBox::new(&data.guess)
            .align_text_middle()
            .right_from(ids.button, 10.0)
            .w_h(200.0, 50.0)
            .set(ids.textbox, ui) {
            match edit {
                text_box::Event::Enter => {
                    data.info = game.new_guess(&data.guess);
                }
                text_box::Event::Update(text) => {
                    data.new_guess(&text);
                }
            }
        }

        Text::new(&data.info)
            .middle_of(ids.textbox)
            .right_from(ids.textbox, 10.0)
            .font_size(25)
            .set(ids.info_text, ui);
    }
}
```
I'd like you to have some fun with positioning the elements and coming up with your own layout.
But furthermore **extend the application** with and `else {/*new game and set new range*/}` branch! You can check out my 
solution.

### 6. Theming - TODO
