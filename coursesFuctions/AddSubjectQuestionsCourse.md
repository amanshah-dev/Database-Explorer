-- Create function to add subject questions for a course
create function "AddSubjectQuestionsCourse"(subjectid integer, courseid integer) returns void
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
    PERFORM "AddChapterQuestionsCourse"(rec."chapterId", courseId);
  END LOOP;
END
$$;

-- Alter function owner
alter function "AddSubjectQuestionsCourse"(integer, integer) owner to learner;
