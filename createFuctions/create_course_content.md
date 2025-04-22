# SQL Function: create_course_content

```sql
create function create_course_content(
    p_course_id integer, 
    original_physics_topic_ids integer[] DEFAULT ARRAY[]::integer[], 
    original_chemistry_topic_ids integer[] DEFAULT ARRAY[]::integer[], 
    original_zoology_topic_ids integer[] DEFAULT ARRAY[]::integer[], 
    original_botany_topic_ids integer[] DEFAULT ARRAY[]::integer[], 
    physics_test_ids integer[] DEFAULT ARRAY[]::integer[], 
    chemistry_test_ids integer[] DEFAULT ARRAY[]::integer[], 
    zoology_test_ids integer[] DEFAULT ARRAY[]::integer[], 
    botany_test_ids integer[] DEFAULT ARRAY[]::integer[]) 
returns void 
language plpgsql 
as 
$$
DECLARE
    physics_id INT;
    chemistry_id INT;
    zoology_id INT;
    botany_id INT;
    topic_name TEXT;
    test_id INT;
    topic_index INT;
    original_topic_id INT;
    topic_id INT;
    course_record "Course"%ROWTYPE;
BEGIN
    -- Add after the course check and before the CourseTest insertions
    if array_length(original_physics_topic_ids, 1) != array_length(physics_test_ids, 1) then
        RAISE EXCEPTION 'Physics topics count (%) does not match physics tests count (%)',
            array_length(original_physics_topic_ids, 1), array_length(physics_test_ids, 1);
    end if;

    if array_length(original_chemistry_topic_ids, 1) != array_length(chemistry_test_ids, 1) then
        RAISE EXCEPTION 'Chemistry topics count (%) does not match chemistry tests count (%)',
            array_length(original_chemistry_topic_ids, 1), array_length(chemistry_test_ids, 1);
    end if;

    if array_length(original_zoology_topic_ids, 1) != array_length(zoology_test_ids, 1) then
        RAISE EXCEPTION 'Zoology topics count (%) does not match zoology tests count (%)',
            array_length(original_zoology_topic_ids, 1), array_length(zoology_test_ids, 1);
    end if;

    if array_length(original_botany_topic_ids, 1) != array_length(botany_test_ids, 1) then
        RAISE EXCEPTION 'Botany topics count (%) does not match botany tests count (%)',
            array_length(original_botany_topic_ids, 1), array_length(botany_test_ids, 1);
    end if;

    SELECT * INTO STRICT course_record FROM "Course" WHERE id = p_course_id;

    if course_record is null then
        RAISE NOTICE 'Course with ID % does not exist', p_course_id;
        return;
    end if;

    -- Link tests to subjects in CourseTest
    FOREACH test_id IN ARRAY physics_test_ids
        LOOP
            BEGIN
                INSERT INTO "CourseTest" ("courseId", "testId")
                VALUES (p_course_id, test_id) on conflict do nothing;
            EXCEPTION
                WHEN foreign_key_violation THEN
                    RAISE NOTICE 'Foreign key violation for test_id: %', test_id;
                    CONTINUE;
            END;
        END LOOP;

    FOREACH test_id IN ARRAY chemistry_test_ids
        LOOP
            BEGIN
                INSERT INTO "CourseTest" ("courseId", "testId")
                VALUES (p_course_id, test_id) on conflict do nothing;
            EXCEPTION
                WHEN foreign_key_violation THEN
                    RAISE NOTICE 'Foreign key violation for test_id: %', test_id;
                    CONTINUE;
            END;
        END LOOP;

    FOREACH test_id IN ARRAY zoology_test_ids
        LOOP
            BEGIN
                INSERT INTO "CourseTest" ("courseId", "testId")
                VALUES (p_course_id, test_id) on conflict do nothing;
            EXCEPTION
                WHEN foreign_key_violation THEN
                    RAISE NOTICE 'Foreign key violation for test_id: %', test_id;
                    CONTINUE;
            END;
        END LOOP;

    FOREACH test_id IN ARRAY botany_test_ids
        LOOP
            BEGIN
                INSERT INTO "CourseTest" ("courseId", "testId")
                VALUES (p_course_id, test_id) on conflict do nothing;
            EXCEPTION
                WHEN foreign_key_violation THEN
                    RAISE NOTICE 'Foreign key violation for test_id: %', test_id;
                    CONTINUE;
            END;
        END LOOP;

    if course_record."hasQuestionBank" = True then
        -- Insert subjects and store their IDs
        INSERT INTO "Subject" ("name", "createdAt", "updatedAt", "courseId")
        VALUES ('Physics', NOW(), NOW(), p_course_id)
        RETURNING id INTO physics_id;

        INSERT INTO "Subject" ("name", "createdAt", "updatedAt", "courseId")
        VALUES ('Chemistry', NOW(), NOW(), p_course_id)
        RETURNING id INTO chemistry_id;

        INSERT INTO "Subject" ("name", "createdAt", "updatedAt", "courseId")
        VALUES ('Zoology', NOW(), NOW(), p_course_id)
        RETURNING id INTO zoology_id;

        INSERT INTO "Subject" ("name", "createdAt", "updatedAt", "courseId")
        VALUES ('Botany', NOW(), NOW(), p_course_id)
        RETURNING id INTO botany_id;


        -- Create topics for Physics
        topic_index := 0;
        FOREACH original_topic_id IN ARRAY original_physics_topic_ids
            LOOP
                topic_index := topic_index + 1;
                select name into topic_name from "Topic" where id = original_topic_id;

                INSERT INTO "Topic" (name, position, "subjectId", "createdAt", "updatedAt", "seqId", published)
                VALUES (topic_name, topic_index, physics_id, NOW(), NOW(), topic_index, true) RETURNING id into topic_id;

                INSERT INTO "DuplicateChapter"("origId", "dupId") values (original_topic_id, topic_id);

                insert into "SubjectChapter" ("chapterId", "subjectId", "createdAt", "updatedAt") values (topic_id, physics_id, NOW(), NOW());
                BEGIN
                    insert into "ChapterQuestionSet"("testId", "chapterId", "createdAt", "updatedAt")
                    values (physics_test_ids[topic_index], topic_id, NOW(), NOW());
                EXCEPTION
                    WHEN foreign_key_violation THEN
                        RAISE NOTICE 'Foreign key violation for testId: % and chapterId: %', physics_test_ids[topic_index], topic_id;
                        CONTINUE;
                END;
            END LOOP;

        -- Create topics for Chemistry
        topic_index := 0;
        FOREACH original_topic_id IN ARRAY original_chemistry_topic_ids
            LOOP
                topic_index := topic_index + 1;
                select name into topic_name from "Topic" where id = original_topic_id;

                INSERT INTO "Topic" (name, position, "subjectId", "createdAt", "updatedAt", "seqId", published)
                VALUES (topic_name, topic_index, chemistry_id, NOW(), NOW(), topic_index, true) RETURNING id into topic_id;

                INSERT INTO "DuplicateChapter"("origId", "dupId") values (original_topic_id, topic_id);

                insert into "SubjectChapter" ("chapterId", "subjectId", "createdAt", "updatedAt") values (topic_id, chemistry_id, NOW(), NOW());
                BEGIN
                    insert into "ChapterQuestionSet"("testId", "chapterId", "createdAt", "updatedAt")
                    values (chemistry_test_ids[topic_index], topic_id, NOW(), NOW());
                EXCEPTION
                    WHEN foreign_key_violation THEN
                        RAISE NOTICE 'Foreign key violation for testId: % and chapterId: %', chemistry_test_ids[topic_index], topic_id;
                        CONTINUE;
                END;
            END LOOP;

        -- Create topics for Zoology
        topic_index := 0;
        FOREACH original_topic_id IN ARRAY original_zoology_topic_ids
            LOOP
                topic_index := topic_index + 1;
                select name into topic_name from "Topic" where id = original_topic_id;

                INSERT INTO "Topic" (name, position, "subjectId", "createdAt", "updatedAt", "seqId", published)
                VALUES (topic_name, topic_index, zoology_id, NOW(), NOW(), topic_index, true) RETURNING id into topic_id;

                INSERT INTO "DuplicateChapter"("origId", "dupId") values (original_topic_id, topic_id);

                insert into "SubjectChapter" ("chapterId", "subjectId", "createdAt", "updatedAt") values (topic_id, zoology_id, NOW(), NOW());
                BEGIN
                    insert into "ChapterQuestionSet"("testId", "chapterId", "createdAt", "updatedAt")
                    values (zoology_test_ids[topic_index], topic_id, NOW(), NOW());
                EXCEPTION
                    WHEN foreign_key_violation THEN
                        RAISE NOTICE 'Foreign key violation for testId: % and chapterId: %', zoology_test_ids[topic_index], topic_id;
                        CONTINUE;
                END;
            END LOOP;

        -- Create topics for Botany
        topic_index := 0;
        FOREACH original_topic_id IN ARRAY original_botany_topic_ids
            LOOP
                topic_index := topic_index + 1;
                select name into topic_name from "Topic" where id = original_topic_id;

                INSERT INTO "Topic" (name, position, "subjectId", "createdAt", "updatedAt", "seqId", published)
                VALUES (topic_name, topic_index, botany_id, NOW(), NOW(), topic_index, true) RETURNING id into topic_id;

                INSERT INTO "DuplicateChapter"("origId", "dupId") values (original_topic_id, topic_id);

                insert into "SubjectChapter" ("chapterId", "subjectId", "createdAt", "updatedAt") values (topic_id, botany_id, NOW(), NOW());
                BEGIN
                    insert into "ChapterQuestionSet"("testId", "chapterId", "createdAt", "updatedAt")
                    values (botany_test_ids[topic_index], topic_id, NOW(), NOW());
                EXCEPTION
                    WHEN foreign_key_violation THEN
                        RAISE NOTICE 'Foreign key violation for testId: % and chapterId: %', botany_test_ids[topic_index], topic_id;
                        CONTINUE;
                END;
            END LOOP;
    end if;

END
$$;

alter function create_course_content(integer, integer[], integer[], integer[], integer[], integer[], integer[], integer[], integer[]) owner to learner;
