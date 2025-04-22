create function "SyncSubjectQuestionsCourse"(subjectid integer, courseid integer) returns void
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
    PERFORM "SyncChapterQuestionsCourse"(rec."chapterId", courseid);
  END LOOP;
END
$$;

alter function "SyncSubjectQuestionsCourse"(integer, integer) owner to learner;
