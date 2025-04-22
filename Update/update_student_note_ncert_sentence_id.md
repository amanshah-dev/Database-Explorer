create function update_student_note_ncert_sentence_id() returns trigger
    language plpgsql
as
$$
BEGIN
    -- Check if "startDataUid" and "endDataUid" are not null
    IF NEW."startDataUid" IS NOT NULL AND NEW."endDataUid" IS NOT NULL THEN
        -- Check for matching row in "NcertSentence"
        UPDATE "StudentNote"
        SET "ncertSentenceId" = ns."id"
        FROM "NcertSentence" ns
        WHERE ns."noteId" = NEW."noteId"
          AND ns."startUid" = NEW."startDataUid"
          AND ns."endUid" = NEW."endDataUid"
          AND "StudentNote"."id" = NEW."id";
    END IF;

    RETURN NEW;
END;
$$;

alter function update_student_note_ncert_sentence_id() owner to neetprep_rw;
