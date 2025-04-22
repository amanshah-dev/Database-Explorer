create function update_tests() returns void
    language plpgsql
as
$$
DECLARE
    test RECORD;
BEGIN
    FOR test IN
        SELECT * FROM "Test"
        WHERE id IN (
            1963151, 1963286, 2092324, 2081381, 1974358, 2057423, 2066132, 1968809, 1984745, 2241034, 2853502,
            1982446, 1936242, 1988466, 1951406, 1947280, 1949250, 1933927, 1932113, 1933897, 1945522, 1932067,
            1933131, 1932487, 1994269, 1958530, 1970928, 1977535, 1979136, 1974502, 1977315, 2561646, 2403857, 3821116
        )
    LOOP
        -- Update duration and sections based on test name
        IF test."name" LIKE '%NEET PG%' THEN
            -- NEET PG pattern: 40 questions per section
            UPDATE "Test"
            SET "durationInMin" = (test."numQuestions" / 40) * 42 + MOD(test."numQuestions", 40),
                "sections" = public.populate_sections(test."numQuestions", 'NEET PG'),
                "lockSection" = true,
                "updatedAt" = NOW()
            WHERE id = test.id;
        ELSIF test."name" LIKE '%AIIMS%' OR test."name" LIKE '%INICET%' THEN
            -- AIIMS/INICET pattern: 45 questions per section
            UPDATE "Test"
            SET "durationInMin" = (test."numQuestions" / 50) * 45 + MOD(test."numQuestions", 50),
                "sections" = public.populate_sections(test."numQuestions", 'AIIMS/INICET'),
                "lockSection" = true,
                "updatedAt" = NOW()
            WHERE id = test.id;
        ELSE
            -- Default behavior (if no specific pattern is matched)
            UPDATE "Test"
            SET "durationInMin" = (test."numQuestions" / 40) * 42 + MOD(test."numQuestions", 40),
                "sections" = public.populate_sections(test."numQuestions"),
                "lockSection" = true,
                "updatedAt" = NOW()
            WHERE id = test.id;
        END IF;
    END LOOP;
END;
$$;

alter function update_tests() owner to neetprep_rw;
