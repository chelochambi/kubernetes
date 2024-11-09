# Título
**Negrillas**

_Italica_

>bloque

cp <your_source_directory> <your_destination_directory>

To stop the command, press <kbd>Control</kbd>+<kbd>C</kbd>.

In the **Commit message** box, type `This is my merge request`.

```plaintext
This is codeblock style
```
- Unordered list item 1

  A line nested using 2 spaces to align with the `U` above.

- Unordered list item 2

  > A quote block that will nest
  > inside list item 2.

- Unordered list item 3

  ```plaintext
  a code block that nests inside list item 3
  ```

- Unordered list item 4

  ![an image that will nest inside list item 4](image.png)

| App name | Description          | Requirements   |
|:---------|:---------------------|:---------------|
| App 1    | Description text 1.  | Requirements 1 |
| App 2    | Description text 2.  | None           |


| App name | Description                      |check |
|:---------|:---------------------------------|------|
| App A    | Description text. <sup>1</sup>   |**{dotted-circle}** No|
| App B    | Description text. <sup>2</sup>   |**{check-circle}** si|

1. This is the footnote.
1. This is the other footnote.



- Item 1
- Item 2
- Item 3
   - Sub item 1
   - Sub item 2
- Item 4


1. Item one
   1. Sub item one
   1. Sub item two
1. Item two
1. Item three


```plantuml
!define ICONURL https://raw.githubusercontent.com/tupadr3/plantuml-icon-font-sprites/v2.1.0
skinparam defaultTextAlignment center
!include ICONURL/common.puml
!include ICONURL/font-awesome-5/gitlab.puml
!include ICONURL/font-awesome-5/java.puml
!include ICONURL/font-awesome-5/rocket.puml
!include ICONURL/font-awesome/newspaper_o.puml
FA_NEWSPAPER_O(news,good news!,node) #White {
FA5_GITLAB(gitlab,GitLab.com,node) #White
FA5_JAVA(java,PlantUML,node) #White
FA5_ROCKET(rocket,Integrated,node) #White
}
gitlab ..> java
java ..> rocket
```
*strong importance* (aka bold)
_stress emphasis_ (aka italic)
`monospaced` (aka typewriter text)
"`double`" and '`single`' typographic quotes
+passthrough text+ (substitutions disabled)
`+literal text+` (monospaced with substitutions disabled)


**C**reate+**R**ead+**U**pdate+**D**elete
fan__freakin__tastic
``mono``culture


[[idname,reference text]]
// or written using normal block attributes as `[#idname,reftext=reference text]`
A paragraph (or any block) with an anchor (aka ID) and reftext.

See <<idname>> or <<idname,optional text of internal link>>.

xref:document.adoc#idname[Jumps to anchor in another document].

This paragraph has a footnote.footnote:[This is the text of the footnote.]
