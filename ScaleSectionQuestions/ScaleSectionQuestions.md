create function "ScaleSectionQuestions"(input_distribution chapter_section_questions[], testpatternid integer) 
    returns chapter_section_questions[] 
    language plpgsql 
as 
$$
DECLARE 
    element chapter_section_questions; 
    total_target_questions INTEGER; 
    section_totals INTEGER[] := '{}'; -- Array to store total questions per section index 
    section_targets INTEGER[] := '{}'; -- Array to store target questions per section index 
    section_names section_type[] := '{}'; -- Array to store section names for indexing 
    section_index INTEGER; 
    results_array chapter_section_questions[] := ARRAY[]::chapter_section_questions[]; -- Initialize empty array of custom type 
    section_chapter_questions chapter_section_questions[] := ARRAY[]::chapter_section_questions[]; -- Temporary storage for questions per section 
BEGIN 
    -- Initialize the totals for each section based on input distribution 
    FOREACH element IN ARRAY input_distribution LOOP 
        section_index := array_position(section_names, element.section_name); 
        IF section_index IS NULL THEN 
            -- New section encountered, expand the arrays 
            section_names := array_append(section_names, element.section_name); 
            section_totals := array_append(section_totals, element.num_questions); 
            section_targets := array_append(section_targets, 0); 
        ELSE 
            -- Existing section, update the total 
            section_totals[section_index] := section_totals[section_index] + element.num_questions; 
        END IF; 
    END LOOP; 

    -- Retrieve the target number of questions for each section from TestPatternSection and store it 
    FOR section_index IN 1..array_length(section_names, 1) LOOP 
        SELECT "numQuestions" INTO total_target_questions 
        FROM "TestPatternSection" 
        WHERE "section" = section_names[section_index] 
        AND "testPatternId" = testPatternId; 

        section_targets[section_index] := total_target_questions; 
    END LOOP; 

    -- Scale the distribution according to the targets 
    FOR section_index IN 1..array_length(section_names, 1) LOOP 
        section_chapter_questions := ARRAY[]::chapter_section_questions[]; 
        FOREACH element IN ARRAY input_distribution LOOP 
            IF element.section_name = section_names[section_index] THEN 
                -- Calculate new question count and append to output array 
                section_chapter_questions := array_append(section_chapter_questions, ROW( 
                    element.chapter_id, 
                    element.section_name, 
                    ROUND(element.num_questions::NUMERIC * section_targets[section_index] / section_totals[section_index])::INTEGER 
                )::chapter_section_questions); 
            END IF; 
        END LOOP; 

        -- Adjust the distribution to match the target 
        section_chapter_questions := "AdjustNumQuestions"(section_chapter_questions, section_targets[section_index]); 

        -- Append the adjusted questions to the result array 
        FOREACH element IN ARRAY section_chapter_questions LOOP 
            results_array := array_append(results_array, element); 
        END LOOP; 

    END LOOP; 

    RETURN results_array; 
END; 
$$;

alter function "ScaleSectionQuestions"(chapter_section_questions[], integer) owner to neetprep_rw;
