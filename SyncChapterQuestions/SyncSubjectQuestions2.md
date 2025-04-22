create function "SyncSubjectQuestions"(subjectid integer) returns void
    language plpgsql
as
$$
DECLARE 
  rec RECORD;
BEGIN 
  FOR rec IN 
    SELECT "chapterId" from "SubjectChapter", "Subject" 
      where "SubjectChapter"."subjectId" = "Subject"."id"
        and "Subject"."id" = subjectid
  LOOP
    PERFORM "SyncChapterQuestions"(rec."chapterId");
  END LOOP;
END
$$;

alter function "SyncSubjectQuestions"(integer) owner to learner;
