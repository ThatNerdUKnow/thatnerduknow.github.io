+++
title = "Making an Enigma Machine"
date = "2022-10-24T16:53:41-05:00"
author = "Brandon Piña"
authorTwitter = "" #do not include @
cover = ""
tags = ["rust", "project","programming"]
keywords = ["rust", "project","programming"]
description = "Using the rust programming language, I create a multi-threaded CLI app that emulates an Enigma Machine that's way faster than it needs to be"
showFullContent = false
readingTime = true
hideComments = false
+++

## What's an Enigma Machine?
> The Enigma machine is a cipher device developed and used in the early- to mid-20th century to protect commercial, diplomatic, and military communication. It was employed extensively by Nazi Germany during World War II, in all branches of the German military. The Enigma machine was considered so secure that it was used to encipher the most top-secret messages. -- Wikipedia
## How does it work?
The Enigma machine is comprised of 3 components:
- The Rotor Mechanism comprised of 3 rotors from a set of Five originally but more rotor variants were added later
- A Reflector
- A plugboard

Each component effectively works as a ***Substitution Cipher*** or ***Caesar Cipher***. It's one of the simplest ciphers there is, Simply put, each character is substituted with another character and no characters map to themselves (because that would render the cipher pointless); Think of Caesar Ciphers as those secret decoder ring toys you give to kids so that they can pretend that they're spies.  

Now if the Enigma machine is implemented using simple Caesar Ciphers, what made it so tough to crack? The answer lies in the rotor mechanism. Every time a character is encoded through the machine, the first rotor advances in position, and in certain positions there are notches, which allow the next rotor in the sequence to advance. The machine's configuration changes as you type! What this means in practice is that if You were to encode the same character multiple times you would get different characters every time.

A property of the enigma machine, Enabled by use of the reflector allows messages encoded by the machine for a given configuration to also be decoded by the machine. If you know the configuration of the machine for an encoded message, you can decode the ciphertext and get the original plaintext back

## Let's get rusty
Feel free to skip this part, I get very detailed here
### The Cipher newtype
Since every component is implemented as a Caesar Cipher, it makes sense to start creating a type that each of our components can use internally, so that we don't have to re-write encoding and decoding logic. The perfect underlying structure for a caesar cipher is a HashMap, or rather, a *bidirectional* HashMap because the mapping relationship between characters goes both ways

```rust
pub struct Cipher(
    HashMap<Character, Character, BuildNoHashHasher<Character>>,
    HashMap<Character, Character, BuildNoHashHasher<Character>>,
);
```
As you can see, I implemented the `Cipher` struct as a newtype over a tuple of HashMaps; one for each side of the mapping relationship. I feel as if I should explain what's going on in the type arguments here. The first type argument for HashMap is the type for the **Key** and the second type argument is the type for the **Value**. The third argument is a bit special. Internally HashMaps in rust use a cryptographically secure hashing function to generate hashes for its keys. The reason for this is to prevent DOS attacks against the hash. This is fine and all, but it's rather slow for our purposes. Fortunately for us, HashMap allows us to replace the Hashing Function that it uses internally. Since the data we're storing in this hashmap isn't terribly important and the data inside the `Character` newtype (more on that later) is a simple char we can use the `NoHash` hashing function. NoHash does exactly what it sounds like, Nothing. If my understanding is correct, it uses the bitwise representation of the value itself as its key. Since it's not doing very much work under the hood, this makes the hashing function much, much faster. While this is much faster than using the original hashing function, It was only marginally (2%) faster than the original implementation which used a Vector Internally. I would say the reason for this is likely because *n* is small and the relatively expensive operation of decoding is the only place where we would have seen a speed-up and that only happens three times per Character in the entire machine.

The Cipher type includes a bit of validation inside the constructor
```rust
impl FromStr for Cipher {
    type Err = Bruh;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        // Convert each char in the string slice into the Character newtype
        // Ensuring that the only characters inside the provided string are
        // A-Z
        let res: Vec<Character> = s
            .chars()
            .into_iter()
            .map(|c| Character::try_from(c))
            .try_collect()
            .with_context(|| format!("Tried to create a cipher from a string"))?;

        // Check the length of the provided string, we need exactly 26 characters
        match s.len() {
            0..=25 => Err(bruh!(CipherError::TooFew(s.len()))),
            26 => Cipher::try_from(res), // Try to construct Cipher from Vec of Characters
            _ => Err(bruh!(CipherError::TooMany(s.len()))),
        }
    }
}

impl TryFrom<Vec<Character>> for Cipher {
    type Error = Bruh;

    fn try_from(value: Vec<Character>) -> Result<Self, Self::Error> {
        // Remove duplicates from provided Vector
        let res = value.iter().unique().count();
        match res {
            // If we get 26 unique Characters, we can start building our Cipher
            // We iterate through chars A-Z then convert them into Character newtypes
            // We use the current value of the iterator to generate our mapping
            26 => ('A'..='Z')
                .into_iter()
                .map(|c| Character::try_from(c).unwrap())
                .enumerate()
                .fold(Ok(Cipher::new()), |acc, (i, next)| match acc {
                    Ok(mut cipher) => {
                        cipher.0.insert(next, value[i]); // Map next->value[i]
                        cipher.1.insert(value[i], next); // Map value[i]->next
                        Ok(cipher) // Return the valid cipher
                    }
                    Err(_) => unreachable!(),
                }),
            _ => Err(bruh!(CipherError::Unique)),
        }
    }
}
```

As you can see the validation rules implemented for Cipher are
- Any string passed must be exactly 26 characters long
- Each string must contain exactly 26 unique characters
- Each string must only contain characters A-Z

### The Character and Position newtypes

#### What's a newtype?
**In Gundam:** *A newtype is an evolved species of humans that have adapted to life in space*   
**In Programming:** *A newtype is a type declared from the definition of an existing type*  

In the context of programming, what does this really mean?

Our language gives us primitive types that we can use to compose data structures. These primitives typically include booleans (true and false), numerical types (integers, floats), string slices and probably a few i'm forgetting. Let's say that I want to represent a distance as a floating point decimal. Let's say kilometers and meters. I might go about it in this way:
```rust
let kilometers: f64 = 10;
let meters: f64 = 100;
```
Can you see the problem? `kilometers` and `meters` are both the same type! That means that adding them together is perfectly valid! Also since they're both `f64`'s they can both be initialized as negative. We don't want negative distances, that's impossible! In the newtype pattern what we would do instead is wrap each of our scalars in a *named tuple* Then impose validation rules inside functions that construct this type.

```rust
pub struct Kilometers(f64);
pub struct Meters(f64);

// Use your imagination for the constructors
// This tangent is long enough
```
We could also implement the arithmetic operators between these types, allowing us to add, subtract, divide and multiply variables of these types and get results that we expect! Keep in mind that the newtype pattern is much more than simple *type aliasing* It's essentially a named tuple!

#### Rant over

In libenigma I defined three newtypes; one of which i've already explained. The remaining newtypes are Character and Position.
Character is used throughout the project while Position is used exclusively inside the Rotor Struct

```rust
#[derive(Debug, PartialEq, Eq, Hash, Clone, Copy)]
pub struct Character(char);

// Implementing this trait allows us to use it as a key in NoHash
impl IsEnabled for Character {} 

#[derive(PartialEq, Eq, Clone, Copy, Debug, Hash, PartialOrd)]
pub struct Position(u8);

```

The constructor for Character ensures that the internal char is only characters between 'A' and 'Z'. It does this using a match statement. If the character matches to the range match arm 'A'..='Z' it is allowed through, otherwise it returns an error type with the original char wrapped inside (this is important so that we don't lose symbols, or non alphabetic characters). The constructor for Position is virtually identical, so i'll omit it
```rust
impl TryFrom<char> for Character {
    type Error = ParsingError;

    fn try_from(value: char) -> Result<Self, Self::Error> {
        let value_uppercase = value.to_ascii_uppercase(); // Convert the char to uppercase
        match value_uppercase {
            'A'..='Z' => Ok(Character(value_uppercase)),
            _ => Err(ParsingError::Charset(value)),
        }
    }
}
```

### The Rotor Struct
The rotor struct contains the following fields
```rust
pub struct Rotor {
    position: Position,
    cipher: Cipher,
    notches: Notches,
}

// I lied, there's more than three newtypes, but this one isn't public
// I should rather say that there's  three publicly exposed newtypes
#[derive(Hash, Debug)]
struct Notches(Vec<Position>); 
```

Each Cipher contains two functions, *encode* and *decode*. We use these methods to share the encoding logic between each component of the enigma machine.
The logic in these functions is deceptively simple, what you don't see is a whole mess of well-tested overloaded operators
```rust
 fn encode_at(&self, c: Character, n: usize) -> Character {
        let offset: Position = self.position + n;
        self.cipher.encode(c + offset)
    }

    fn decode_at(&self, c: Character, n: usize) -> Character {
        let offset: Position = self.position + n;
        let dec = self.cipher.decode(c);
        dec - offset
    }
```
There's actually a lot to unpack here. These functions gave me a lot of trouble when it came around to integration testing because for the life of me I coulnd't figure out why i could encode a character through a rotor, then decode the ciphertext back through the rotor and i'd get a different character from what I started out with. THe problem was *where* I was doing the addidion of the rotor's offset to the encoding. In encode_at I was adding the offset to the character before encoding the character while originally I was doing the same in decode_at. However, it took some thinking through and talking to a rubber duck to reason around why this was causing me problems. Unfortunately I lack the language skills to explain why

Also the reason these functions are called encode_at instead of simply encode is that here we're encoding the character given a certain number of rotations of the current rotor. This is absolutely critical for multithreading later as we don't have to rely on internal state, meaning that we can encode characters *out of order* but we have to be able to predict how many times the current rotor has advanced. That's where this function comes in:
```rust
/// given n revolutions of the current rotor, how many times will the next rotor in the sequence advance?
    fn get_num_advances(&self, n: usize) -> usize {
        let r = n / 26; // Number of full revolutions
        let notches_left = self
            .notches
            .0
            .iter()
            .filter(|notch| self.position <= **notch)
            .count(); // How many notches are left in the first rotation of the rotor, given the inital position of the rotor

        let final_position = Position::try_from((n % 26) as u8).unwrap(); // gets the position of the rotor after n advances
        let notches_past = self
            .notches
            .0
            .iter()
            .filter(|notch| final_position > **notch)
            .count(); // How many notches have we passed in the current partial revolution

        // Multiply full rotations by the number of notches and add the number of notches left in the first rotation
        let mut result = (r * self.notches.0.len()) + notches_left; 

        // If we've completed at least one rotation, add notches_past
        if r > 0 {
            result = result + notches_past
        }

        result
    }
```
This function says "Given how many times this rotor has advanced, how many times will it advance the next rotor?". We can use this information to pass to the next rotor in the sequence to get the position of every rotor given *n* characters encoded. 

Also, I don't allow construction of Rotors directly, I generate them based off of the reference table from wikipedia, Meaning that assuming that my unit tests are correct and wikipedia's information is correct, you could decipher real world enigma transmissions using this library. Neat!
```rust

#[derive(
    EnumString, EnumIter, Hash, PartialEq, Eq, Clone, Copy, Display, Debug, Serialize, Deserialize,
)]
pub enum Rotors {
    I,
    II,
    III,
    IV,
    V,
    VI,
    VII,
    VIII,
}

impl TryFrom<(Rotors, char)> for Rotor {
    type Error = Bruh;

    fn try_from((variant, position): (Rotors, char)) -> Result<Self, Self::Error> {
        match variant {
            Rotors::I => Rotor::new("EKMFLGDQVZNTOWYHXUSPAIBRCJ", &['Q'], position),
            Rotors::II => Rotor::new("AJDKSIRUXBLHWTMCQGZNPYFVOE", &['E'], position),
            Rotors::III => Rotor::new("BDFHJLCPRTXVZNYEIWGAKMUSQO", &['V'], position),
            Rotors::IV => Rotor::new("ESOVPZJAYQUIRHXLNFTGKDCMWB", &['J'], position),
            Rotors::V => Rotor::new("VZBRGITYUPSDNHLXAWMJQOFECK", &['Z'], position),
            Rotors::VI => Rotor::new("JPGVOUMFYQBENHZRDKASXLICTW", &['Z', 'M'], position),
            Rotors::VII => Rotor::new("NZJHGRCXMYSWBOUFAIVLPEKQDT", &['Z', 'M'], position),
            Rotors::VIII => Rotor::new("FKQHTLXOCBJSPDZRAMEWNIUYGV", &['Z', 'M'], position),
        }
    }
}

```

### The Reflector
This component was the simplest because It requires little configuration on the part of the user, Simply pick a reflector variant and you're good to go.
The construction of the reflector is virtually identical to Rotor, so i'll omit it but keep in mind that we don't have to keep track of the position of the reflector because it's not a moving part. Otherwise The reflector works exactly the same. However, the reflector has a special property that it only accepts connections on one side. Meaning that insead of being able to encode then decode a character to get your original character back, you are able to encode, the *encode again* to get your original character back. This is the reason why you can decode enigma messages given the same configuration.

Also the encode function is very nice to look at
```rust
impl Encode for Reflector {
    /// Encodes a given char through the reflector.
    /// Given the properties of the reflector, if the output of this function was fed through this function again,
    /// you would get back the original char
    fn encode(&self, c: Character) -> Character {
        self.cipher.encode(c)
    }
}
```

### The plugboard
In the enigma machine you are given ten plugs, You are given the choice to map any character to any other character. You may not map multiple characters to the same character(because you physically could not on the original machine) and you can use no more than 10 plugs(because the machine only came with 10) you also don't have to use all of the plugs. Or any for that matter. Given the complexity of this component (and the fact i had to write an interface to generate one from the command line) I'll try to keep this short

```rust
pub struct Plugboard {
    cipher: Cipher,
}
```
Internally the Plugboard contains only a cipher. Most of the logic in this module is geared towards *constructing* a valid cipher string given user input
Basically we take a vec of tuples `(Character,Character)` and validate them against the aforementioned invariants then create our cipher string and use it as our internal cipher. Likewise, the encode function is very simple here. We simply pass the given Character to the internal Cipher newtype. Oh also I should mention that like the Reflector, The mappings in the plugboard go both ways.

### Putting it together: Enigma
As I said before, each component in the engima machine acts as a substitution cipher. However, the order in which you apply these ciphers is important
- A Key is pressed
- That character is encoded through the plugboard
- The ciphertext from the plugboard is then encoded through each individual rotor
- The output from the rotor mechanism is encoded through the reflector
- The output from the reflector is fed *backwards* through the rotor mechanism
- The ciphertext makes one final pass through the plugboard
- Then we get the final character

Here's what the code looks like for a single character:
```rust
fn encode_at(&self, c: Character, n: usize) -> Character {
        let plugboard_enc = self.plugboard.encode(c);
        let rotor_enc = self.rotors.encode_at(plugboard_enc, n);
        let reflector_enc = self.reflector.encode(rotor_enc);
        let rotor_dec = self.rotors.decode_at(reflector_enc, n);
        let plugboard_dec = self.plugboard.decode(rotor_dec);
        plugboard_dec
    }
```

Oh my, I lied again! I actually declared *four* public newtypes. Essentially here I abstract away feeding the output of each rotor into the next into a single operation
```rust
pub struct RotorConfig(Vec<Rotor>);

impl RotorConfig {
    pub fn encode_at(&self, c: Character, n: usize) -> Character {
        let encode_first_rotor = self.0[0].encode_at(c, n);
        let n = self.0[0].get_num_advances(n);
        let encode_second_rotor = self.0[1].encode_at(encode_first_rotor, n);
        let n = self.0[1].get_num_advances(n);
        let encode_third_rotor = self.0[2].encode_at(encode_second_rotor, n);
        encode_third_rotor
    }

    pub fn decode_at(&self, c: Character, n: usize) -> Character {
        let r1_advances = n;
        let r2_advances = self.0[0].get_num_advances(n);
        let r3_advances = self.0[1].get_num_advances(r2_advances);

        let decode_third_rotor = self.0[2].decode_at(c, r3_advances);
        let decode_second_rotor = self.0[1].decode_at(decode_third_rotor, r2_advances);
        let decode_first_rotor = self.0[0].decode_at(decode_second_rotor, r1_advances);
        decode_first_rotor
    }
}
```

It's not terribly useful to only be able to encode one character at a time, So I needed to write a function that could encode an entire String
```rust
pub fn encode(&self, s: &String) -> String {
        s.par_char_indices()
            .map(|(i, c)| (i, Character::try_from(c)))
            .map(|(n, c)| match c {
                Ok(plain) => self.encode_at(plain, n).into(), // If character is 'A'-'Z', encode it
                Err(e) => match e {
                    ParsingError::Charset(non_alphabetic) => non_alphabetic, // Otherwise, simply return the original character
                    _ => unreachable!(),
                },
            })
            .collect()
    }
```
There's actually something special going on here. Can you see it? I'm multithreading it! Using the excellent `rayon` crate, I'm able to generate a *parallel* iterator from this string, allowing me to build my logic in the same manner as if I was writing a single threaded version!!!

## The Benchmarks:
The benchmark generates a string of random characters (All A-Z) given length *n*
I've generated benchmarks for strings of the following lengths
- 1 thousand
- 10 thousand
- 100 thousand
- 1 million
- 10 million
- 100 million

Here's the results


```
     Running benches\perf.rs (target\release\deps\perf-f5e366b5f8262b43.exe)
1k                      time:   [215.39 µs 216.71 µs 218.12 µs]
Found 5 outliers among 100 measurements (5.00%)
  5 (5.00%) high mild

10k                     time:   [591.08 µs 594.62 µs 598.26 µs]
Found 4 outliers among 100 measurements (4.00%)
  1 (1.00%) low mild
  3 (3.00%) high mild

Benchmarking 100k: Warming up for 3.0000 s
Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 9.6s, enable flat sampling, or reduce sample count to 50.
100k                    time:   [1.8469 ms 1.8593 ms 1.8730 ms]
Found 3 outliers among 100 measurements (3.00%)
  3 (3.00%) high mild

1m                      time:   [9.3270 ms 9.4011 ms 9.4863 ms]
Found 1 outliers among 100 measurements (1.00%)
  1 (1.00%) high severe

Benchmarking 10m: Warming up for 3.0000 s
Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 7.9s, or reduce sample count to 60.
10m                     time:   [79.100 ms 79.525 ms 79.963 ms]
Found 4 outliers among 100 measurements (4.00%)
  4 (4.00%) high mild

Benchmarking 100m: Warming up for 3.0000 s
Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 78.5s, or reduce sample count to 10.
100m                    time:   [768.39 ms 771.83 ms 775.81 ms]
Found 5 outliers among 100 measurements (5.00%)
  2 (2.00%) high mild
  3 (3.00%) high severe

```
That's right, We can encode 100 Million characters in less than a second, Or more practically, 1 million characters in around 9 Milliseconds. To put that in perspective, Herman Melville's *Moby Dick* utf-8 version from project gutenburg is a little over a megabyte. If we assume that each character takes up between 1 and 4 bytes per character (and let's face it it's mostly ascii) that gives us approximately a little over a million characters. We could encode the entirety of **Moby Dick** In ***Thousandths*** of a second. How about Crime and Punishment? War and peace? 
- Crime and Punishment: 1.1MB
- War and Peace: 3.2 MB  
Not even Russian Literature stands a chance!