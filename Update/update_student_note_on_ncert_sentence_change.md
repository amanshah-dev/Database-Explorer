create function update_student_note_on_ncert_sentence_change() returns trigger
    language plpgsql
as
$$
BEGIN
    -- Update the "StudentNote" table if the corresponding "NcertSentence" is updated
    UPDATE "StudentNote"
    SET "ncertSentenceId" = NEW."id",
        "updatedAt" = NOW()
    WHERE "noteId" = NEW."noteId"
      AND "ncertSentenceId" IS NULL 
      AND "startDataUid" IS NOT NULL 
      AND "endDataUid" IS NOT NULL 
      AND "noteId" IS NOT NULL 
      AND "startDataUid" = NEW."startUid"
      AND "endDataUid" = NEW."endUid";
    
    RETURN NEW;
END;
$$;

alter function update_student_note_on_ncert_sentence_change() owner to neetprep_rw;
