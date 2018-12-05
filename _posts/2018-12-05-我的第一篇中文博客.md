## ?Sized
```
struct Unknow_Size<T: ?Sized> {
    s: T;
}

let str_size = Unknow_Size{s: "jdeng".to_string()};
let vec_size = Unknow_Size{s: vec![1,2,3]};
```

## Box
```
```

## Cell
```
```

## char to string
```
let a = ['a', 'b', 'c'];
let b: String = a.into_iter().collect(); // "abc"
```

## str to int
```
use std::str::FromStr;

let u: u32 = Fromstr::from_str("123").unwrap(); // 123

// or use parse
let u: u32 = "123".parse::<u32>().unwrap()
```

## Primitives

Type | i32 | u32 | String | f64
---|---|---|---|---
i32 | n/a | x as u32 | x.to_string() | x as f64
u32 | x as i32 | n/a | x.to_string() | x as f64
String* | x.parse::<i32>().unwrap() | x.parse::<u32>().unwrap() | n/a | x.parse::<f64>().unwrap()
f64 | x as i32 | x as u32 | x.to_string() | n/a

## check data type
```
fn typeid<T: std::any::Any>(_: &T) {
    println!("{:?}", std::any::TypeId::of::<T>());
}

use std::any::{Any, TypeId};

fn is_string<T: ?Sized + Any>(_s: &T) -> bool {
    TypeId::of::<String>() == TypeId::of::<T>()
}

fn main() {
    assert_eq!(is_string(&0), false);
    assert_eq!(is_string(&"cookie monster".to_string()), true);
}
```

## naked pointer
```
let x = 5;
let raw = &x as *const i32;
let mut y = 10;
let raw_mut = &mut y as *mut i32; // mut pointer

let x = 5;
let raw = &x as *const i32;
let points_at = unsafe { *raw }; // must use unsafe to deref pointer
println!("raw points at {}", points_at);
```