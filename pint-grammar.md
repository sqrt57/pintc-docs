# Pint Formal Grammar

Grammar for Pint v0.7, in Go-style EBNF.
See [pint-spec-v0.7.md](pint-spec-v0.7.md) for the full language specification.

---

## Notation

Each production has the form:

    Rule = expression .

| Operator | Meaning |
|---|---|
| `\|` | alternation |
| `( )` | grouping |
| `[ ]` | zero or one |
| `{ }` | zero or more |
| `"text"` | literal terminal |
| `` `text` `` | literal terminal containing `"` |
| `a … b` | any character in the inclusive range a through b |
| `/* prose */` | terminal described informally |

---

## Lexical Grammar

Source files are UTF-8 encoded. Whitespace (U+0020, U+0009, U+000D, U+000A)
and comments are skipped between tokens.

### Characters

    newline        = /* U+000A */ .
    unicode_char   = /* any Unicode scalar value except U+000A */ .
    ascii_char     = /* any Unicode scalar value in the range U+0000 … U+007F */ .
    letter         = "a" … "z" | "A" … "Z" | "_" .
    decimal_digit  = "0" … "9" .
    hex_digit      = "0" … "9" | "a" … "f" | "A" … "F" .
    binary_digit   = "0" | "1" .

### Comments

    line_comment  = "//" { unicode_char } newline .
    block_comment = "/*" { unicode_char | newline } "*/" .

Block comments are not nested.

### Identifiers

    identifier = letter { letter | decimal_digit } .

Reserved keywords — may not be used as identifiers:

    and    bool    break  byte   const  continue  else    enum
    export extern  false  f32    f64    for       fun     i8
    i16    i32     i64    if     import isize     loop    module
    nil    not     or     record return string    true    type
    usize  u8      u16    u32    u64    var       while   xor

Predefined names — may not be shadowed by user declarations:

    cast  divmod  length  mul  sizeof  to_i8  to_i16  to_i32  to_u8  to_u16  to_u32

`_` is the discard identifier; it is reserved and may not be used as a declared name.

### Integer Literals

    integer_lit = decimal_lit | hex_lit | binary_lit .
    decimal_lit = decimal_digit { decimal_digit } .
    hex_lit     = "0x" hex_digit { hex_digit } .
    binary_lit  = "0b" binary_digit { binary_digit } .

### Float Literals

    float_lit = decimal_digit { decimal_digit } "." decimal_digit { decimal_digit } [ exponent ] .
    exponent  = ( "e" | "E" ) [ "+" | "-" ] decimal_digit { decimal_digit } .

At least one digit is required on each side of the decimal point.

### String Literals

    string_lit     = `"` { string_char | escape_seq | unicode_escape } `"` .
    string_char    = /* any unicode_char except `"` and `\` */ .
    escape_seq     = `\"` | `\'` | `\\` | `\n` | `\r` | `\t` | `\0` .
    unicode_escape = `\u{` hex_digit { hex_digit } `}` .

### Character Literals

    char_lit        = "'" ( char_char | char_escape_seq ) "'" .
    char_char       = /* any ascii_char in the range U+0000 … U+007F except `'` and `\` */ .
    char_escape_seq = `\"` | `\'` | `\\` | `\n` | `\r` | `\t` | `\0` .

Character literals have type `byte`. Only ASCII code points (0x00–0x7F) are valid.
`\u{...}` is not valid in character literals.

---

## Syntactic Grammar

### Source File

    SourceFile = { Module } .
    Module     = "module" identifier "{" { ModuleDecl } "}" .
    ModuleDecl = FunDecl
               | ModuleVarDecl
               | ConstDecl
               | RecordDecl
               | EnumDecl
               | TypeDecl
               | ExternDecl
               | ExportDecl
               | ImportDecl
               .

### Types

    Type = PrimitiveType
         | identifier
         | "^" Type
         | "[" Expr "]" Type
         | "(" ")"
         | "(" [ TypeList ] ")" "->" ReturnType
         .

    PrimitiveType = "i8"  | "i16"  | "i32"  | "i64"
                  | "u8"  | "u16"  | "u32"  | "u64"
                  | "f32" | "f64"
                  | "bool" | "byte" | "usize" | "isize" | "string" .

    TypeList   = Type { "," Type } .
    ReturnType = "(" ReturnItem { "," ReturnItem } ")"
               | Type
               .
    ReturnItem = [ identifier ":" ] Type .

`"(" ")"` is the unit type. `"(" [ TypeList ] ")" "->" ReturnType` is a function type.
Either all `ReturnItem`s carry a name or none do — a semantic constraint, not syntactic.

### Attributes

    Attribute   = "[" AttrBody "]" .
    AttrBody    = identifier "(" AttrArgList ")"
               | identifier "=" AttrValue
               | identifier
               .
    AttrArgList = AttrArg { "," AttrArg } .
    AttrArg     = identifier "=" AttrValue .
    AttrValue   = integer_lit | string_lit | identifier .

### Module-Scope Declarations

    FunDecl       = { Attribute } "fun" identifier "(" [ ParamList ] ")" "->" ReturnType Block .
    ExternDecl    = { Attribute } "extern" "fun" identifier "(" [ ParamList ] ")" "->" ReturnType ";" .
    ParamList     = Param { "," Param } .
    Param         = identifier ":" Type .

    ModuleVarDecl = "var" identifier ":" Type [ "=" Expr ] ";" .
    ConstDecl     = "const" identifier ":" Type "=" Expr ";" .
    TypeDecl      = "type" identifier "=" Type ";" .

    RecordDecl    = "record" identifier { Attribute } "{" { FieldDecl } "}" .
    FieldDecl     = { Attribute } identifier ":" Type "," .

    EnumDecl      = "enum" identifier [ ":" PrimitiveType ] "{" { EnumVariant } "}" .
    EnumVariant   = identifier [ "=" Expr ] "," .

    ImportDecl    = "import" identifier "as" identifier ";" .
    ExportDecl    = "export" ExportItem { "," ExportItem } ";" .
    ExportItem    = identifier [ "(" ExportFields ")" ] .
    ExportFields  = "*" | identifier { "," identifier } .

### Statements

    Block = "{" { Stmt } "}" .
    Stmt  = VarStmt
          | ConstDecl
          | AssignStmt
          | MultiAssignStmt
          | ExprStmt
          | IfStmt
          | ForStmt
          | WhileStmt
          | LoopStmt
          | ReturnStmt
          | BreakStmt
          | ContinueStmt
          | Block
          .

    VarStmt      = "var" ( SingleVar | MultiVar ) ";" .
    SingleVar    = identifier ":" Type [ "=" Expr ] .
    MultiVar     = "(" MultiVarItem { "," MultiVarItem } ")" "=" Expr .
    MultiVarItem = ( identifier ":" Type ) | "_" .

    AssignStmt      = Expr "=" Expr ";" .                                    (* LHS must be assignable *)
    MultiAssignStmt = "(" MultiAssignItem { "," MultiAssignItem } ")" "=" Expr ";" .
    MultiAssignItem = ( identifier ":" identifier ) | identifier | "_" .
    ExprStmt        = Expr ";" .

    IfStmt    = "if" "(" Expr ")" Block [ "else" ( IfStmt | Block ) ] .
    ForStmt   = [ Label ] "for" "(" ForInit ";" Expr ";" ForPost ")" Block .
    WhileStmt = [ Label ] "while" "(" Expr ")" Block .
    LoopStmt  = [ Label ] "loop" Block .
    Label     = identifier ":" .
    ForInit   = "var" identifier ":" Type "=" Expr .
    ForPost   = Expr "=" Expr .                                              (* LHS must be assignable *)

    ReturnStmt   = "return" [ Expr { "," Expr } ] ";" .
    BreakStmt    = "break" [ identifier ] ";" .
    ContinueStmt = "continue" [ identifier ] ";" .

### Expressions

Operators in decreasing precedence order: postfix, unary, `*`/`/`/`%`, `+`/`-`,
`<<`/`>>`, `<`/`>`/`<=`/`>=`, `==`/`!=`, `&`, `xor`, `|`, `and`, `or`.

    Expr           = OrExpr .
    OrExpr         = AndExpr { "or" AndExpr } .
    AndExpr        = BitwiseOrExpr { "and" BitwiseOrExpr } .
    BitwiseOrExpr  = XorExpr { "|" XorExpr } .
    XorExpr        = BitwiseAndExpr { "xor" BitwiseAndExpr } .
    BitwiseAndExpr = EqualityExpr { "&" EqualityExpr } .
    EqualityExpr   = CompareExpr { ( "==" | "!=" ) CompareExpr } .
    CompareExpr    = ShiftExpr { ( "<" | ">" | "<=" | ">=" ) ShiftExpr } .
    ShiftExpr      = AddExpr { ( "<<" | ">>" ) AddExpr } .
    AddExpr        = MultExpr { ( "+" | "-" ) MultExpr } .
    MultExpr       = UnaryExpr { ( "*" | "/" | "%" ) UnaryExpr } .
    UnaryExpr      = PostfixExpr
                   | "-"   UnaryExpr
                   | "~"   UnaryExpr
                   | "not" UnaryExpr
                   | "@"   UnaryExpr
                   .
    PostfixExpr    = PrimaryExpr { PostfixOp } .
    PostfixOp      = "^"
                   | "[" Expr "]"
                   | "." identifier
                   | "->" identifier
                   | "(" [ ArgList ] ")"
                   .
    PrimaryExpr    = integer_lit
                   | float_lit
                   | string_lit
                   | char_lit
                   | "true" | "false" | "nil"
                   | identifier
                   | "(" Expr ")"
                   | RecordLit
                   | ArrayLit
                   | CastExpr
                   | SizeofExpr
                   .

    RecordLit  = "{" FieldInit { "," FieldInit } "}" .
    FieldInit  = identifier ":" Expr .
    ArrayLit   = "[" Expr { "," Expr } "]" .
    CastExpr   = "cast"   "(" Type "," Expr ")" .
    SizeofExpr = "sizeof" "(" Type ")" .

    ArgList        = PositionalArgs | NamedArgs .
    PositionalArgs = Expr { "," Expr } .
    NamedArgs      = NamedArg { "," NamedArg } .
    NamedArg       = identifier ":" Expr .
