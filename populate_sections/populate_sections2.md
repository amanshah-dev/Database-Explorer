create function populate_sections(num_questions integer, test_name text, section_num_qs integer DEFAULT 40) returns json
    language plpgsql
as
$$
DECLARE
    sections jsonb := '[]'::jsonb;
    i integer;
    full_sections integer;
    last_section_questions integer;
    question_counter integer := 1;
    effective_section_num_qs integer := section_num_qs;
BEGIN
    -- Determine the number of questions per section based on the test name
    IF test_name ILIKE '%Neet PG%' THEN
        effective_section_num_qs := 40;
    ELSIF test_name ILIKE '%AIIMS%' OR test_name ILIKE '%INICET%' THEN
        effective_section_num_qs := 50;
    END IF;

    -- Calculate the number of full sections and questions in the last section
    full_sections := num_questions / effective_section_num_qs;
    last_section_questions := num_questions % effective_section_num_qs;

    -- If numQuestions <= effective_section_num_qs, only one section "Part-A" starting from question 1
    IF num_questions <= effective_section_num_qs THEN
        sections := jsonb_build_array(jsonb_build_array('Part-A', question_counter)::jsonb);
    ELSE
        -- Create full sections with starting question number
        FOR i IN 0 .. full_sections - 1 LOOP
            sections := sections || jsonb_build_array(jsonb_build_array('Part-' || chr(65 + i), question_counter)::jsonb);
            question_counter := question_counter + effective_section_num_qs; -- Move to the next starting question number
        END LOOP;

        -- Add the last section with the remaining questions
        IF last_section_questions > 0 THEN
            sections := sections || jsonb_build_array(jsonb_build_array('Part-' || chr(65 + full_sections), question_counter)::jsonb);
        END IF;
    END IF;

    -- Return the generated sections as a JSON object
    RETURN sections::json;
END;
$$;

alter function populate_sections(integer, text, integer) owner to neetprep_rw;
