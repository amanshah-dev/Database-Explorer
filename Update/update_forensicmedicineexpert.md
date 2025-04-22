create procedure update_forensicmedicineexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Forensic Medicine and Toxicology', achievementCategoryId, userIds);
END;
$$;

alter procedure update_forensicmedicineexpert(bigint, bigint[]) owner to learner;
