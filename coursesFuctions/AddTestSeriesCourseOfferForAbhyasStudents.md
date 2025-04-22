-- Create function to adjust the number of questions in each chapter section
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

-- Alter function owner
alter function "AdjustNumQuestions"(chapter_section_questions[], integer) owner to neetprep_rw;
