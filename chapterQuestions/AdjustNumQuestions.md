# Function: `AdjustNumQuestions`

## Description:
This function adjusts the number of questions in each chapter section so that the total number of questions matches a given target. If the total number of questions is less than the target, the function adds questions to the chapter sections as evenly as possible. If the total number of questions exceeds the target, it removes questions from the chapter sections. The adjustments are made iteratively and as evenly as possible across the available chapter sections.

## Parameters:
- `chapter_questions` (array of `chapter_section_questions`): An array of records, each containing the current number of questions in a chapter section. The function will adjust these numbers to meet the target.
- `target_num_questions` (integer): The target number of questions that should be assigned across the chapter sections.

## Returns:
- `chapter_section_questions[]`: An array of adjusted `chapter_section_questions`, where the number of questions for each chapter section is updated to match the target total.

## Logic:
1. **Calculate Total Current Questions**:
   The function first calculates the total number of questions currently assigned across all chapter sections by summing the `num_questions` in the `chapter_questions` array.

2. **Calculate the Difference**:
   The function calculates the difference (`diff`) between the target number of questions (`target_num_questions`) and the current total (`total_current_questions`).

3. **No Adjustment Needed**:
   If the difference is `0`, meaning the current total already matches the target, the function simply returns the input array without any modifications.

4. **Adjust Questions**:
   If adjustment is needed, the function distributes the change across the chapter sections:
   - If the target is greater than the current total, it adds questions to the chapter sections.
   - If the target is smaller than the current total, it removes questions from the chapter sections.
   - The adjustments are made as evenly as possible across the chapter sections by iterating through the array and adding or removing one question at a time.

5. **Return Adjusted Array**:
   After making the necessary adjustments, the function returns the updated `chapter_questions` array.

## SQL Code:

```sql
create function "AdjustNumQuestions"(chapter_questions chapter_section_questions[], target_num_questions integer) returns chapter_section_questions[]
    language plpgsql
as
$$
DECLARE
    total_current_questions INTEGER;
    diff INTEGER;
    adjust_index INTEGER;
    extra_questions INTEGER;
    i INTEGER;
BEGIN
    -- Initialize total_current_questions
    total_current_questions := 0;

    -- Calculate the total number of questions currently assigned
    IF array_length(chapter_questions, 1) IS NOT NULL THEN
        FOR i IN 1..array_length(chapter_questions, 1) LOOP
            total_current_questions := total_current_questions + chapter_questions[i].num_questions;
        END LOOP;
    END IF;

    -- Calculate the difference between the target and current total
    diff := target_num_questions - total_current_questions;

    -- If no adjustment is needed, return the input array as is
    IF diff = 0 THEN
        RETURN chapter_questions;
    END IF;

    -- Adjust questions as evenly as possible
    adjust_index := 1;
    extra_questions := ABS(diff);
    WHILE extra_questions > 0 LOOP
        IF diff > 0 THEN
            -- Increase num_questions
            chapter_questions[adjust_index].num_questions := chapter_questions[adjust_index].num_questions + 1;
            extra_questions := extra_questions - 1;
        ELSE
            -- Decrease num_questions
            IF diff < 0 AND chapter_questions[adjust_index].num_questions >= 1 THEN
                chapter_questions[adjust_index].num_questions := chapter_questions[adjust_index].num_questions - 1;
                extra_questions := extra_questions - 1;
            END IF;
        END IF;
        adjust_index := adjust_index + 1;
        IF adjust_index > array_length(chapter_questions, 1) THEN
            adjust_index := 1;
        END IF;
    END LOOP;

    RETURN chapter_questions;
END;
$$;

alter function "AdjustNumQuestions"(chapter_section_questions[], integer) owner to neetprep_rw;
