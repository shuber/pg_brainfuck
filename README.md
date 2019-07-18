# pg_brainfuck

* https://en.wikipedia.org/wiki/Brainfuck
* https://esolangs.org/wiki/Brainfuck_implementations

```sql
-- Hello, World!
select brainfuck('+[-->-[>>+>-----<<]<--<---]>-.>>>+.>>..+++[.>]<<<<.+++.------.<<-.>>>>+.');

-- test from stdin
select brainfuck(',[.,]', 'test from stdin'::bytea);
select brainfuck(',[.[-],]', 'test from stdin'::bytea);
select brainfuck(',[>,]<[.<]', 'nidts morf tset'::bytea);
```

```sql
create or replace function brainfuck(
  in program text,
  in stdin bytea default ''::bytea,
  out stdout text
) as $$
  declare
    commands text[] := regexp_split_to_array(program, '');
    commands_size int := array_upper(commands, 1);

    cells int[] := array[]::int[];
    jumps int[] := array[]::int[];
    stack int[] := array[]::int[];

    depth int;
    index int := 1;
    pointer int := 0;
    read int := 0;
    subindex int;
  begin
    while index <= commands_size loop
      case commands[index]
      when '>' then
        pointer = pointer + 1;
      when '<' then
        pointer = pointer - 1;
      when '+' then
        cells[pointer] = coalesce(cells[pointer], 0) + 1;
        cells[pointer] = coalesce(nullif(cells[pointer], 256), 0);
      when '-' then
        cells[pointer] = coalesce(cells[pointer], 0) - 1;
        cells[pointer] = coalesce(nullif(cells[pointer], -1), 255);
      when '.' then
        continue when coalesce(cells[pointer], 0) = 0;
        stdout = coalesce(stdout, '') || chr(cells[pointer]);
      when ',' then
        begin
          cells[pointer] = get_byte(stdin, read);
          read = read + 1;
        exception when array_subscript_error then
          index = index + 1;
        end;
      when ']' then
        if array_length(stack, 1) > 0 then
          subindex = stack[array_upper(stack, 1)];
          stack = array_remove(stack, subindex);
          jumps[subindex] = index;
          index = subindex - 1;
        end if;
      when '[' then
        if coalesce(cells[pointer], 0) != 0 then
          stack = array_append(stack, index);
        elsif jumps[index] is not null then
          index = jumps[index];
        else
          depth = 1;
          index = index + 1;
          subindex = index;

          while subindex <= commands_size loop
            case commands[subindex]
            when '[' then depth = depth + 1;
            when ']' then depth = depth - 1;
            else end case;

            if depth = 0 then
              jumps[index] = subindex;
              index = subindex;
              exit;
            end if;

            subindex = subindex + 1;
          end loop;
        end if;
      else end case;

      index = index + 1;
    end loop;
  end;
$$ language plpgsql immutable;
```
