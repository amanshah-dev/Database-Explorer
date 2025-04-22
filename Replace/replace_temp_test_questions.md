create function replace_temp_test_questions(temp_test_id integer, backup_temp_test_id integer, question_ids_to_remove integer[]) 
    returns integer[] 
    language plpgsql 
as 
$$ 
DECLARE 
    question_id INTEGER; 
    chapter_id INTEGER; 
    section_name section_type; 
    seq_num INTEGER; 
    new_question_id INTEGER; 
    new_question_ids INTEGER[] := ARRAY[]::INTEGER[]; 
BEGIN 
    -- Loop over each question in the list of question_ids_to_remove 
    FOREACH question_id IN ARRAY question_ids_to_remove 
    LOOP 
        -- Step 1: Mark the question as deleted in TempTest 
        SELECT "chapterId", "section", "seqNum" INTO chapter_id, section_name, seq_num 
        FROM "TempTestQuestion" 
        WHERE "tempTestId" = temp_test_id AND "questionId" = question_id; 

        -- Step 2: Find a suitable replacement from BackupTempTest by chapter and section 
        SELECT ttt."questionId" INTO new_question_id 
        FROM "TempTestQuestion" ttt 
        WHERE ttt."tempTestId" = backup_temp_test_id 
          AND ttt."chapterId" = chapter_id 
          AND ttt."section" = section_name 
          AND ttt."deleted" = false 
          AND ttt."questionId" NOT IN ( 
              SELECT "questionId" 
              FROM "TempTestQuestion" 
              WHERE "tempTestId" = temp_test_id 
          ) 
        ORDER BY ttt."questionId" 
        LIMIT 1; 

        -- Step 3: If no question is found, find a replacement by section only 
        IF new_question_id IS NULL THEN 
            SELECT ttt."questionId" INTO new_question_id 
            FROM "TempTestQuestion" ttt 
            WHERE ttt."tempTestId" = backup_temp_test_id 
              AND ttt."section" = section_name 
              AND ttt."deleted" = false 
              AND ttt."questionId" NOT IN ( 
                  SELECT "questionId" 
                  FROM "TempTestQuestion" 
                  WHERE "tempTestId" = temp_test_id 
              ) 
            ORDER BY ttt."questionId" 
            LIMIT 1; 
        END IF; 

        -- Step 4: Insert the new question into TempTestQuestion at the specific seqNum 
        IF new_question_id IS NOT NULL THEN 
            -- Step 5: Mark the new question as deleted in BackupTempTest 
            UPDATE "TempTestQuestion" 
            SET "deleted" = true, 
                "updatedAt" = NOW() 
            WHERE "tempTestId" = backup_temp_test_id AND "questionId" = new_question_id; 

            UPDATE "TempTestQuestion" 
            SET "deleted" = true, 
                "updatedAt" = NOW() 
            WHERE "tempTestId" = temp_test_id AND "questionId" = question_id; 

            INSERT INTO "TempTestQuestion" ("tempTestId", "questionId", "chapterId", "section", "seqNum", "createdAt", "updatedAt") 
            SELECT temp_test_id, new_question_id, chapter_id, section_name, seq_num, NOW(), NOW() 
            WHERE NOT EXISTS ( 
                SELECT 1 
                FROM "TempTestQuestion" 
                WHERE "tempTestId" = temp_test_id 
                  AND "questionId" = new_question_id 
            ); 

            -- Step 6: Add the new question ID to the array 
            new_question_ids := array_append(new_question_ids, new_question_id); 
        END IF; 
    END LOOP; 

    -- Return the list of newly replaced question IDs 
    RETURN new_question_ids; 

END; 
$$;

alter function replace_temp_test_questions(integer, integer, integer[]) owner to neetprep_rw;
